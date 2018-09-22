---


---

<p>I have been really excited to use <a href="https://kubernetes.io/">Kubernetes</a> (aka k8s) in a semi-production way on my home server; however, there is only one of said server and not the 3+ that seem to be typical of a minimal kubernetes deployment. At the same time, I know there’s <a href="https://github.com/kubernetes/minikube/">minikube</a> and similar, but those give a strong sense of development-time, experimental usage. I want a small scale, but <em>real</em> kubernetes experience.</p>
<p>For me, an additional requirement to make it a real deployment is that it should support stateful services. One of the things I like about kubernetes is that <a href="https://kubernetes.io/docs/concepts/storage/persistent-volumes/">persistent volumes</a> are a first-class construct supported in various real-world ways. Since I’m working with a tight budget and a single server, I narrowed my search to open source, software-defined storage solutions. <a href="https://ceph.com/">Ceph</a> workeds out well for me since it was easy to setup, scaled down without compromises, and is full of featurea.</p>
<p>In this article I’ll show how to use a little libvirt+KVM+QEMU wrapper script to create three VMs, deploy kubernetes using <code>kube-adm</code> and overlay a ceph cluster using <code>ceph-deploy</code>. The setup might appear tedious, but the benefits and ease of kubernetes usage afterwards are well worth it</p>
<h2 id="my-environment">My environment</h2>
<p>If you’re following along, it might help to know what I’m using to see how that aligns with what you’re using. My one and only home server is an <a href="https://www.intel.com/content/www/us/en/products/boards-kits/nuc.html">Intel NUC</a> with</p>
<ul>
<li>i5 dual-core, hyperthreaded processor</li>
<li>16 GB RAM</li>
<li>240 GB SSD with about 100 GB dedicated to VM images</li>
<li>Ubuntu Xenial 16.04.4</li>
<li>Kernel 4.4.0-124</li>
</ul>
<p>It has plenty of RAM for three VMs, but I’m going to be knowingly overcommitting the vCPU count of 6 (2 x 3 VMs) since there’s only 4 logical processors on my system. I’m not going to be running stressful workloads, so hopefully that works out.</p>
<h2 id="setup-virtual-machines">Setup virtual machines</h2>
<h3 id="baremetal-or-vms">Baremetal or VMs?</h3>
<p>My original desire was to avoid virtual machines (VMs) entirely and run both kubernetes and ceph directly “on metal”. I found that was technically possible with options such as “untainting” the k8s master node; however, striving for zero downtime really requires distinct nodes in order to achieve seamless cluster scheduling.</p>
<p>As such, one of the wheels I re-invented was to create a small tool to help me create VMs with minimal requirements. All that it requires is installation of the packages:</p>
<ul>
<li>libvirt-bin</li>
<li>qemu-kvm</li>
</ul>
<h3 id="networking-setup-for-virtualization">Networking setup for virtualization</h3>
<p>Since I’ll eventually be port forwarding certain traffic from my ISP’s cable modem, I need the VMs to be directly routable on my LAN. For that I needed to configure the Linux kernel bridging by <a href="https://wiki.libvirt.org/page/Networking#Altering_the_interface_config">following this</a>.</p>
<p>In my case, I set my <code>/etc/network/interfaces</code> with the following since I am using my LAN’s DHCP server to assign a fixed IP address. So the <code>dhcp</code> option eliminates the need specify address, gateway, and DNS.</p>
<pre><code>auto br0
iface br0 inet dhcp
        bridge_ports eno1
        bridge_maxwait 0
        bridge_fd 0
</code></pre>
<p>To make sure iptable filtering and such of the host doesn’t interfere with the guest VMs doing their own, we ease packet forwarding by creating <code>/etc/sysctl.d/60-bridge-virt.conf</code> with</p>
<pre><code>net.bridge.bridge-nf-call-arptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
</code></pre>
<p>To apply those settings immediately, run:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> /lib/systemd/systemd-sysctl --prefix<span class="token operator">=</span>/net/bridge
</code></pre>
<p>To ensure the settings are applied on reboot after networking is started, then create the file <code>/etc/udev/rules.d/99-bridge.rules</code> with the content:</p>
<pre><code>ACTION=="add", SUBSYSTEM=="net", KERNEL!="lo", RUN+="/lib/systemd/systemd-sysctl --prefix=/net/bridge"
</code></pre>
<h3 id="create-vms-using-libvirtkvmqemu">Create VMs using libvirt+KVM+QEMU</h3>
<p>Download my helper script <a href="https://github.com/itzg/libvirt-tools">from my libvirt-tools repo</a> and set it to be executable:</p>
<pre><code>wget https://raw.githubusercontent.com/itzg/libvirt-tools/master/create-vm.sh
chmod +x create-vm.sh
</code></pre>
<p>I am going to create three VMs. My network has static IP address space starting at 192.168.0.150, so I’m configuring my VMs starting from that address.</p>
<p>The volume on my machine that contains <code>/var/lib/libvirt/images</code> on my system only has 112G so I am going to allocate an extra disk device of 20G for the three VMs. In total they will use 3 x (8G root + 20G extra) = 84G. The volumes are thin provisioned, but it’s still good to plan for maximum usage since I have limited host volume space (SSD == good for speed, but bad for capacity).</p>
<p>View full usage and default values of the helper script by running</p>
<pre class=" language-bash"><code class="prism  language-bash">./create-vms.sh --help
</code></pre>
<p>Make sure you have an SSH key setup since the helper script will tell cloud-init to install your current user’s ssh key for access as the user <code>ubuntu</code>. To confirm, list your keys using:</p>
<pre class=" language-bash"><code class="prism  language-bash">ssh-keygen -l
</code></pre>
<p>Create the first VM:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> ./create-vm.sh --ip-address 192.168.0.150 --extra-volume 20G nuc-vm1
</code></pre>
<p>You should now be able to ssh to the VM as the user <code>ubuntu</code></p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">ssh</span> ubuntu@192.168.0.150
</code></pre>
<p>however, if that doesn’t seem to be working you can attach to the VM’s console using:</p>
<pre class=" language-bash"><code class="prism  language-bash">virsh console nuc-vm1
</code></pre>
<p><em>Use Control-] to detach from the console.</em></p>
<p>Repeat the same invocation for the other two VMs changing the IP address and name:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> ./create-vm.sh --ip-address 192.168.0.151 --extra-volume 20G nuc-vm2
<span class="token function">sudo</span> ./create-vm.sh --ip-address 192.168.0.152 --extra-volume 20G nuc-vm3
</code></pre>
<h3 id="simplify-ssh-access">Simplify ssh access</h3>
<p>Adding <code>/etc/hosts</code> entries for the VMs and their IPs will help ease the remainder of the setup tasks. In my case I added these:</p>
<pre><code>192.168.0.150 nuc-vm1
192.168.0.151 nuc-vm2
192.168.0.152 nuc-vm3
</code></pre>
<p>To avoid having to specify the default <code>ubuntu</code> user for ssh’ing to each VM, add this, replacing the host names with yours, to <code>~/.ssh/config</code>:</p>
<pre><code>Host nuc-vm1 nuc-vm2 nuc-vm3
   User ubuntu
</code></pre>
<p>Confirm you can ssh into each, which also gives you a chance to confirm and accept the host fingerprint for later steps:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">ssh</span> nuc-vm1
</code></pre>
<p>While you’re in there you could also upgrade packages to get the VMs up to date before installing more stuff:</p>
<pre><code>sudo apt update
sudo apt upgrade -y
# ...and sudo reboot if the kernel was included in the upgraded packages
</code></pre>
<h2 id="create-kubernetes-cluster">Create kubernetes cluster</h2>
<h3 id="pre-flight-checks">Pre-flight checks</h3>
<p>ssh to the first VM and <a href="https://kubernetes.io/docs/tasks/tools/install-kubeadm/">install kubeadm and its prerequisites</a>.</p>
<p>The VMs were configured (via defaults) to generate unique MAC addresses for their bridged network device, but here’s an example of confirming:</p>
<pre><code>$ ssh nuc-vm1 ip link | grep "link/ether"
    link/ether 52:54:00:91:76:e5 brd ff:ff:ff:ff:ff:ff
$ ssh nuc-vm2 ip link | grep "link/ether"
    link/ether 52:54:00:93:4c:83 brd ff:ff:ff:ff:ff:ff
$ ssh nuc-vm3 ip link | grep "link/ether"
    link/ether 52:54:00:b7:88:ad brd ff:ff:ff:ff:ff:ff
</code></pre>
<p>Likewise, the VMs were created with the default behavior of generating a UUID per domain/node. Here is confirmation of that:</p>
<pre><code>$ ssh nuc-vm1 sudo cat /sys/class/dmi/id/product_uuid
781526BF-31E7-4339-8424-6B886A432968
$ ssh nuc-vm2 sudo cat /sys/class/dmi/id/product_uuid
AF93665F-6137-4125-9480-08B99B040BE8
$ ssh nuc-vm3 sudo cat /sys/class/dmi/id/product_uuid
C76F05E7-0289-4479-91CA-3D47A48096F4
</code></pre>
<h3 id="install-docker-and-kubernetes-packages">Install Docker and Kubernetes packages</h3>
<p>Install the recommended version of Docker by ssh’ing into the first node and starting an interactive sudo session with</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> -i
</code></pre>
<p>Use the installation snippet provided in the <a href="https://kubernetes.io/docs/tasks/tools/install-kubeadm/#installing-docker">Installing Docker</a> section.</p>
<p>Still in the interactive sudo session, <a href="https://kubernetes.io/docs/tasks/tools/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl">install kubeadm, kubelet, and kubectl</a>.</p>
<p>Repeat the same for the other nodes <a href="https://kubernetes.io/docs/tasks/tools/install-kubeadm/#installing-docker">installing Docker</a> and steps after that.</p>
<h3 id="create-the-cluster">Create the cluster</h3>
<p><strong>Before</strong> running <code>kubeadm init</code> skip to the <a href="https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network">pod network section</a> to see what parameters should be passed. I’m going to use kube-router, so I’ll pass <code>--pod-network-cidr=10.244.0.0/16</code>.  You can find more information about using <a href="https://github.com/cloudnativelabs/kube-router/blob/master/docs/kubeadm.md">kube-router with kubeadm here</a>.</p>
<p>The full kubeadm command to run is:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> kubeadm init --pod-network-cidr<span class="token operator">=</span>10.244.0.0/16
</code></pre>
<p>Use the commands it provides at the end to enable kubectl access from the regular <code>ubuntu</code> user on the VM:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">mkdir</span> -p <span class="token variable">$HOME</span>/.kube
<span class="token function">sudo</span> <span class="token function">cp</span> -i /etc/kubernetes/admin.conf <span class="token variable">$HOME</span>/.kube/config
<span class="token function">sudo</span> <span class="token function">chown</span> <span class="token variable"><span class="token variable">$(</span><span class="token function">id</span> -u<span class="token variable">)</span></span><span class="token keyword">:</span><span class="token variable"><span class="token variable">$(</span><span class="token function">id</span> -g<span class="token variable">)</span></span> <span class="token variable">$HOME</span>/.kube/config
</code></pre>
<p>Be sure to note the <code>kubeadm join</code> command it provided since it will be used when joining the other two VMs to the cluster.</p>
<p>Now you can install the networking add-on, kube-router in this case:</p>
<pre class=" language-bash"><code class="prism  language-bash">kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
</code></pre>
<p>After about 30 seconds you should see the <code>kube-router</code> pods running using:</p>
<pre class=" language-bash"><code class="prism  language-bash">kubectl get pods --all-namespaces -w
</code></pre>
<h3 id="join-the-other-two-vms-to-the-kubernetes-cluster">Join the other two VMs to the kubernetes cluster</h3>
<p>ssh to the next VM and become root with <code>sudo -i</code>. Then use the <code>kubectl join</code> command output from the <code>init</code> call earlier, such as</p>
<pre class=" language-bash"><code class="prism  language-bash">kubeadm <span class="token function">join</span> 192.168.0.150:6443 --token b2pp7j<span class="token punctuation">..</span><span class="token punctuation">..</span> --discovery-token-ca-cert-hash sha256:<span class="token punctuation">..</span>.
</code></pre>
<p>If you ssh back to the first node, you should see the new node become ready after about 30 seconds:</p>
<pre class=" language-bash"><code class="prism  language-bash">$ kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
nuc-vm1   Ready     master    22h       v1.10.3
nuc-vm2   Ready     <span class="token operator">&lt;</span>none<span class="token operator">&gt;</span>    23s       v1.10.3
</code></pre>
<p>Repeat the <code>kubeadm join</code> on the third VM.</p>
<h3 id="create-a-distinct-user-config">Create a distinct user config</h3>
<p>On the master node’s VM I ran the following and saved its output to a file on my desktop:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> kubeadm alpha phase kubeconfig user --client-name itzg
</code></pre>
<p>By default that user can’t do much of anything, but you can quickly fix that by creating a cluster role binding to the builtin <code>cluster-admin</code> role. In my case I ran:</p>
<pre class=" language-bash"><code class="prism  language-bash">kubectl create clusterrolebinding itzg-cluster-admin --clusterrole<span class="token operator">=</span>cluster-admin --user<span class="token operator">=</span>itzg
</code></pre>
<p>I used a naming convention of <code>&lt;user&gt;-&lt;role&gt;</code>, but you can use whatever naming convention you want for the first argument – it just needs to distinctly name the binding of user(s) to role(s).</p>
<h2 id="create-ceph-cluster">Create ceph cluster</h2>
<p>Now we’ll switch gears and ignore the fact that the three VMs are participating in a kubernetes cluster. We’ll overlay a ceph cluster onto them. My goal is to create a storage pool dedicated to kubernetes that can be used to provision rbd volumes.</p>
<p>In the same spirit as <code>kubeadm</code> ceph provides an extremely useful tool called <code>ceph-deploy</code> that will take care of setting up our VMs as a ceph cluster. It also comes with <a href="http://docs.ceph.com/docs/master/start/">great instructions, that start here</a>.</p>
<p>At the time of this writing, the <a href="http://docs.ceph.com/docs/master/releases/">latest stable release</a> is <code>luminous</code>, so that’s what I’ll be using in the next few steps.</p>
<p>Most of the steps I’ll run with <code>ceph-deploy</code> will be invoked from the main/baremetal host, but really it can be done from any node that can ssh to the three VMs.</p>
<h3 id="setup-ceph-deploy">Setup ceph-deploy</h3>
<p>First add the release key</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">wget</span> -q -O- <span class="token string">'https://download.ceph.com/keys/release.asc'</span> <span class="token operator">|</span> <span class="token function">sudo</span> apt-key add -
</code></pre>
<p>Then, add the repository:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token keyword">echo</span> deb https://download.ceph.com/debian-luminous/ <span class="token variable"><span class="token variable">$(</span>lsb_release -sc<span class="token variable">)</span></span> main <span class="token operator">|</span> <span class="token function">sudo</span> <span class="token function">tee</span> /etc/apt/sources.list.d/ceph.list
</code></pre>
<p>Finally, update the repo index and install <code>ceph-deploy</code></p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> apt update
<span class="token function">sudo</span> apt <span class="token function">install</span> ceph-deploy
</code></pre>
<p>The tool is going to place some important config files in the current directory, so it’s a good idea to create a new directory and work within there:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">mkdir</span> ceph-cluster
<span class="token function">cd</span> ceph-cluster
</code></pre>
<p>Before proceeding, make sure you did the <code>~/.ssh/config</code> setup above to configure <code>ubuntu</code> as the default ssh user for accessing the three VMs. Since that user already has password-less sudo access, it’s ready to use for ceph deployment.</p>
<h3 id="initialize-cluster-config">Initialize cluster config</h3>
<p>Initialize the ceph cluster configuration to point at what will be the initial monitor node. Think of the ceph monitor kind of like the kubernetes master/api node.</p>
<pre class=" language-bash"><code class="prism  language-bash">ceph-deploy new nuc-vm1
</code></pre>
<h3 id="install-packages-on-ceph-nodes">Install packages on ceph nodes</h3>
<p>Now, install the ceph packages on all three VMs. I had to pass the <code>--release</code> argument to avoid it defaulting to the prior “jewel” release.</p>
<pre class=" language-bash"><code class="prism  language-bash">ceph-deploy <span class="token function">install</span> --release luminous nuc-vm1 nuc-vm2 nuc-vm3
</code></pre>
<h3 id="create-a-ceph-monitor">Create a ceph monitor</h3>
<p>After a few minutes of installing packages, the initial monitor can be setup:</p>
<pre class=" language-bash"><code class="prism  language-bash">ceph-deploy mon create-initial
</code></pre>
<p>To enable automatic use of <code>ceph</code> on the three nodes, distribute the cluster config using</p>
<pre class=" language-bash"><code class="prism  language-bash">ceph-deploy admin nuc-vm1 nuc-vm2 nuc-vm3
</code></pre>
<h3 id="create-a-ceph-manager">Create a ceph manager</h3>
<p>As of the luminous release, a manager node is required. I’m just going to run that also on the first VM:</p>
<pre class=" language-bash"><code class="prism  language-bash">ceph-deploy mgr create nuc-vm1
</code></pre>
<h3 id="setup-each-node-as-a-ceph-osd">Setup each node as a ceph OSD</h3>
<p>Way back in the beginning of all this, you might remember that I specified an extra volume size of 20GB for each VM. That extra volume is <code>/dev/vdc</code> on each VM and will be used for the OSD on each:</p>
<pre class=" language-bash"><code class="prism  language-bash">ceph-deploy osd create --data /dev/vdc nuc-vm1
ceph-deploy osd create --data /dev/vdc nuc-vm2
ceph-deploy osd create --data /dev/vdc nuc-vm3
</code></pre>
<p>You can confirm the ceph cluster is healthy and contains the three OSDs by running:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">ssh</span> nuc-vm1 <span class="token function">sudo</span> ceph -s
</code></pre>
<p>To enable running ceph commands from the host, copy the admin keyring and config file into the local <code>/etc/ceph</code>:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> <span class="token function">cp</span> ceph.client.admin.keyring ceph.conf /etc/ceph/
</code></pre>
<h2 id="joining-ceph-and-kubernetes">Joining ceph and kubernetes</h2>
<h3 id="create-a-storage-pool">Create a storage pool</h3>
<p>Create the pool with 128 placement groups, as <a href="http://docs.ceph.com/docs/master/rados/operations/placement-groups/">recommended here</a>:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> ceph osd pool create kube 128
</code></pre>
<p>In order to avoid the libceph error “missing required protocol features” when kubelet mounts the rbd volume apply this adjustment:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> ceph osd crush tunables legacy
</code></pre>
<h3 id="create-and-store-access-keys">Create and store access keys</h3>
<p>Export the ceph admin key and import it as a kubernetes secret:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> ceph auth get client.admin<span class="token operator">|</span><span class="token function">grep</span> <span class="token string">"key = "</span> <span class="token operator">|</span><span class="token function">awk</span> <span class="token string">'{print  <span class="token variable">$3</span>'</span><span class="token punctuation">}</span> <span class="token operator">|</span><span class="token function">xargs</span> <span class="token keyword">echo</span> -n <span class="token operator">&gt;</span> /tmp/secret
kubectl create secret generic ceph-admin-secret \
   --type<span class="token operator">=</span><span class="token string">"kubernetes.io/rbd"</span> \
   --from-file<span class="token operator">=</span>/tmp/secret
</code></pre>
<p>Create a new ceph client for accessing the <code>kube</code> pool specifically and import that as a kubernetes secret:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> ceph auth get-or-create client.kube mon <span class="token string">'allow r'</span> osd <span class="token string">'allow class-read object_prefix rbd_children, allow rwx pool=kube'</span><span class="token operator">|</span><span class="token function">grep</span> <span class="token string">"key = "</span> <span class="token operator">|</span><span class="token function">awk</span> <span class="token string">'{print  <span class="token variable">$3</span>'</span><span class="token punctuation">}</span> <span class="token operator">|</span><span class="token function">xargs</span> <span class="token keyword">echo</span> -n <span class="token operator">&gt;</span> /tmp/secret
kubectl create secret generic ceph-secret \
   --type<span class="token operator">=</span><span class="token string">"kubernetes.io/rbd"</span> \
   --from-file<span class="token operator">=</span>/tmp/secret
</code></pre>
<p>I might be doing something wrong, but I found I had to also save the <code>client.kube</code> keyring on each of the kubelet nodes:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">ssh</span> nuc-vm1 <span class="token function">sudo</span> ceph auth get client.kube -o /etc/ceph/ceph.client.kube.keyring
<span class="token function">ssh</span> nuc-vm2 <span class="token function">sudo</span> ceph auth get client.kube -o /etc/ceph/ceph.client.kube.keyring
<span class="token function">ssh</span> nuc-vm3 <span class="token function">sudo</span> ceph auth get client.kube -o /etc/ceph/ceph.client.kube.keyring
</code></pre>
<h3 id="setup-rbd-provisioner">Setup RBD provisioner</h3>
<p>The out-of-tree RBD provisioner is pre-configured to manage dynamic allocation of RBD persistent volume claims, so download and extract the necessary files:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">wget</span> https://github.com/kubernetes-incubator/external-storage/archive/master.zip
unzip master.zip <span class="token string">"external-storage-master/ceph/rbd/*"</span>
</code></pre>
<p>Go into the <code>deploy</code> directory and apply the provisioner:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">cd</span> external-storage-master/ceph/rbd/deploy
kubectl apply -f rbac
</code></pre>
<h3 id="define-a-storage-class">Define a storage class</h3>
<p>Create a storage class definition file, such as <code>storage-class.yaml</code> containing:</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">kind</span><span class="token punctuation">:</span> StorageClass
<span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> storage.k8s.io/v1
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> rbd
<span class="token key atrule">provisioner</span><span class="token punctuation">:</span> ceph.com/rbd
<span class="token key atrule">parameters</span><span class="token punctuation">:</span>
  <span class="token key atrule">monitors</span><span class="token punctuation">:</span> 192.168.0.150<span class="token punctuation">:</span><span class="token number">6789</span>
  <span class="token key atrule">pool</span><span class="token punctuation">:</span> kube
  <span class="token key atrule">adminId</span><span class="token punctuation">:</span> admin
  <span class="token key atrule">adminSecretName</span><span class="token punctuation">:</span> ceph<span class="token punctuation">-</span>admin<span class="token punctuation">-</span>secret
  <span class="token key atrule">userId</span><span class="token punctuation">:</span> kube
  <span class="token key atrule">userSecretName</span><span class="token punctuation">:</span> ceph<span class="token punctuation">-</span>secret
  <span class="token key atrule">imageFormat</span><span class="token punctuation">:</span> <span class="token string">"2"</span>
</code></pre>
<p>Apply the storage class:</p>
<pre class=" language-bash"><code class="prism  language-bash">kubectl apply -f storage-class.yaml
</code></pre>
<h3 id="try-it-out">Try it out</h3>
<p>Test the storage class by applying a persistent volume claim:</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">kind</span><span class="token punctuation">:</span> PersistentVolumeClaim
<span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> v1
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> claim1
<span class="token key atrule">spec</span><span class="token punctuation">:</span>
  <span class="token key atrule">accessModes</span><span class="token punctuation">:</span>
    <span class="token punctuation">-</span> ReadWriteOnce
  <span class="token key atrule">storageClassName</span><span class="token punctuation">:</span> rbd
  <span class="token key atrule">resources</span><span class="token punctuation">:</span>
    <span class="token key atrule">requests</span><span class="token punctuation">:</span>
      <span class="token key atrule">storage</span><span class="token punctuation">:</span> 1Gi
</code></pre>
<p>Verify that the claim was provisioned and is bound by using</p>
<pre class=" language-bash"><code class="prism  language-bash">kubectl describe pvc claim1
</code></pre>
<p>Let’s test otf the persistent volume backed by ceph rbd by running a little busybox container that just sleeps so that we can <code>exec</code> into it:</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> v1
<span class="token key atrule">kind</span><span class="token punctuation">:</span> Pod
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> ceph<span class="token punctuation">-</span>pod1
<span class="token key atrule">spec</span><span class="token punctuation">:</span>
  <span class="token key atrule">containers</span><span class="token punctuation">:</span>
  <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> ceph<span class="token punctuation">-</span>busybox
    <span class="token key atrule">image</span><span class="token punctuation">:</span> busybox
    <span class="token key atrule">command</span><span class="token punctuation">:</span> <span class="token punctuation">[</span><span class="token string">"sleep"</span><span class="token punctuation">,</span> <span class="token string">"60000"</span><span class="token punctuation">]</span>
    <span class="token key atrule">volumeMounts</span><span class="token punctuation">:</span>
    <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> data
      <span class="token key atrule">mountPath</span><span class="token punctuation">:</span> /data
      <span class="token key atrule">readOnly</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>
  <span class="token key atrule">volumes</span><span class="token punctuation">:</span>
  <span class="token punctuation">-</span> <span class="token key atrule">name</span><span class="token punctuation">:</span> data
    <span class="token key atrule">persistentVolumeClaim</span><span class="token punctuation">:</span>
      <span class="token key atrule">claimName</span><span class="token punctuation">:</span> claim1
</code></pre>
<p>Now exec into the pod using</p>
<pre class=" language-bash"><code class="prism  language-bash">kubectl <span class="token function">exec</span> -it ceph-pod1 sh
</code></pre>
<p>You can <code>cd /data</code> and touch/edit files in that directory, delete the pod, and re-create a new one to confirm the content from the persistent volume claim sticks around.</p>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

