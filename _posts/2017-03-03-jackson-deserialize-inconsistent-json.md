---
layout: post
title:  "Dealing with inconsistent JSON with Jackson deserializer tweaking"

---

## Dealing with inconsistent JSON with Jackson deserializer tweaking

Let's say we have a model class we want to read from JSON:

```
@Data
public class NotWeird {
    String name;
    Details details;

    @Data
    public static class Details {
        int count;
    }
}
```

but we need to deal with varying structure of `details` like this

```
{
  "name": "weird1",
  "details": 5
}
```

and this

```
{
  "name": "weird2",
  "details": {
    "count": 6
  }
}
```

In our fabricated scenario, maybe there was an older version of our JSON structure where details used to be a single field. Or perhaps we're provided a human-friendly shortcut where only setting `count` of a details doesn't required the whole object.

To recreate this scenario and create a solution, let's create a Spring Boot batch-like application that will autowire a Jackson `ObjectMapper` for us.

We'll use these dependencies in our `pom.xml`:

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-web</artifactId>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
<dependency>
	<groupId>org.projectlombok</groupId>
	<artifactId>lombok</artifactId>
	<optional>true</optional>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>
``` 

Even though we're not creating a web app, we need the `org.springframework.http.converter.json.Jackson2ObjectMapperBuilder` provided by `spring-web`. We'll disable the web application context by using `SpringApplicationBuilder`:

```
@SpringBootApplication
public class InconsistentJsonApplication {

	public static void main(String[] args) {
		new SpringApplicationBuilder(InconsistentJsonApplication.class)
				.web(false)
				.run(args);
	}
}
```



Before getting to an application runner, let's establish a properties component:

```
@ConfigurationProperties("app") @Component @Data
public class AppProperties {
    boolean failOnEmptyFileList = true;
    boolean exitWhenFinished = true;
}
```

We'll wire that and the `ObjectMapper` auto configured by Boot into an `ApplicationRunner`:

```
@Slf4j
@Service
public class Loader implements ApplicationRunner {

    private ObjectMapper objectMapper;
    private AppProperties properties;

    @Autowired
    public Loader(ObjectMapper objectMapper, AppProperties properties) {
        this.objectMapper = objectMapper;
        this.properties = properties;
    }
```

In our runner we'll "load" the JSON content by reading them into POJOs, logging, and exiting:

```
@Override
public void run(ApplicationArguments args) throws Exception {
    if (properties.isFailOnEmptyFileList()) {
        Assert.notEmpty(args.getNonOptionArgs(), 
	        "Pass at least one JSON filename on the command line");
    }

    List<NotWeird> objects = args.getNonOptionArgs().stream()
            .map(this::load)
            .collect(Collectors.toList());

    log.info("Loaded: {}", objects);

    if (properties.isExitWhenFinished()) {
        System.exit(0);
    }
}

private NotWeird load(String filename) {
    try {
        final NotWeird obj = objectMapper.readValue(
	        new File(filename), NotWeird.class);

        return obj;
    } catch (IOException e) {
        log.warn("Unable to process file {}", filename, e);
        return null;
    }
}
```

If you run the code at this point, the `readValue` will fail on the first JSON snippet as follows (with newlines added for clarity):

```
com.fasterxml.jackson.databind.JsonMappingException: 
    Can not construct instance of me.itzg.json.model.NotWeird$Details: 
      no int/Int-argument constructor/factory method to deserialize from Number value (5)
      at [Source: weird1.json; line: 3, column: 14] 
      (through reference chain: me.itzg.json.model.NotWeird["details"])
```

There are probably several ways to solve this, but let's use a solution where we can intercept the deserialization of `details` and handle it in a polymorphic/adapting way.

By declaring a `com.fasterxml.jackson.databind.Module` component, Boot will take care of autowiring that into any `ObjectMapper`s :

```
@Component
public class CustomModule extends Module {
    @Override
    public String getModuleName() {
        return "custom";
    }

    @Override
    public Version version() {
        return new Version(1, 0, 0, null, null, null);
    }
```

The meat of our solution involves adding a `com.fasterxml.jackson.databind.deser.BeanDeserializerModifier`:

```
@Override
public void setupModule(SetupContext context) {
    context.addBeanDeserializerModifier(new BeanDeserializerModifier() {
        @Override
        public JsonDeserializer<?> modifyDeserializer(DeserializationConfig config,
                                                      BeanDescription beanDesc,
                                                      JsonDeserializer<?> deserializer) {

            return deserializer;
        }
    });

}
```

Already with those code our application will run, but will still complain about the first JSON snippet since we didn't actually modify the deserializer yet. Let's do that by using `replaceProperty` of `BeanDeserializer`.

That method takes two `SettableBeanProperty` objects, the original and the one to put in its place. Where do we get those? It turns out that we can a) ask the existing deserializer for one and b) derive a new instance from that one.

We'll be good and only tweak the class we want especially since we know that a `BeanDeserializer` is indeed used in that case:

```
public JsonDeserializer<?> modifyDeserializer(DeserializationConfig config,
                                              BeanDescription beanDesc,
                                              JsonDeserializer<?> deserializer) {

    if (NotWeird.class.isAssignableFrom(beanDesc.getBeanClass())) {
		// ...
    }
```

Now we can ask it for the property definition it is using by default for our "details" field:

```
if (NotWeird.class.isAssignableFrom(beanDesc.getBeanClass())) {
    final BeanDeserializer beanDeserializer = (BeanDeserializer) deserializer;

    final SettableBeanProperty property = beanDeserializer.findProperty("details");

	// ...
}
```

With `property` we can derive a customized instance with our deserializer:

```
final SettableBeanProperty property = beanDeserializer.findProperty("details");

final SettableBeanProperty ourProp = 
	property.withValueDeserializer(new DetailsDeserializer());
beanDeserializer.replaceProperty(property, ourProp);
```

All that's left is to implement that `DetailsDeserializer`:

```
public class DetailsDeserializer extends JsonDeserializer<NotWeird.Details> {
    @Override
    public NotWeird.Details deserialize(JsonParser p, DeserializationContext ctxt) 
            throws IOException, JsonProcessingException {
        // ...return one
    }
}
```

As a starting point, we'll read it as an object (which will throw the same exception):

```
public NotWeird.Details deserialize(JsonParser p, DeserializationContext ctxt)
        throws IOException, JsonProcessingException {
    final NotWeird.Details details = 
	    p.getCodec().readValue(p, NotWeird.Details.class);

    return details;
}
```

Let's hope for the best and wrap the default strategy in a try-catch, use the raw parse tree, and try to process the entry as a single numeric value:

```
NotWeird.Details details;
try {
    details = p.getCodec().readValue(p, NotWeird.Details.class);
} catch (JsonMappingException e) {
    final TreeNode treeNode = p.getCodec().readTree(p);
    final JsonToken token = treeNode.asToken();

    details = new NotWeird.Details();

    if (token.isNumeric()) {
		// ...
    } else {
        ctxt.handleUnexpectedToken(NotWeird.Details.class, token, p, 
	        "Unsupported content for details");
    }
```

The given `DeserializationContext` has several "handle*" methods, which are meant for exactly these cases.

Let's optimistically parse it as an integer and set `count` of the details with that value:

```
if (token.isNumeric()) {
    final String str = treeNode.toString();
    try {
        details.setCount(
                Integer.parseInt(str)
        );
    } catch (NumberFormatException e1) {
        ctxt.handleWeirdStringValue(NotWeird.Details.class, str,
                "Could not parse into an integer");
    }
}
```

Finally, with that surgically precise adjustment, the application processes both JSON snippets and produces the log (newlines added for clarity) we wanted:

```
Loaded: [
  NotWeird(name=weird1, details=NotWeird.Details(count=5)), 
  NotWeird(name=weird2, details=NotWeird.Details(count=6))]
```

