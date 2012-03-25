---
layout: post
title: "Using Dropwizard with mongo jackson mapper"
category : development
tags : [dropwizard, mongodb, json, jackson]
---
{% include JB/setup %}

## Why dropwizard?

It seems to be a fun framework for pure JSON services. It is simple to understand, and developed with operations in mind, you need to deploy and monitor the application - as I started out as rather as administrator with this IT stuff, I always prefer a nicely integrated operations solution. Checkout the [dropwizard homepage](http://dropwizard.codahale.com/) for examples, it also features good documentation.

In order to understand this article I expect you to have at least read the [Getting Started](http://dropwizard.codahale.com/getting-started/) guide from dropwizard.

## Why mongo-jackson-mapper?

First checkout [mongo-jackson-mapper](http://vznet.github.com/mongo-jackson-mapper/) - and be inspired by its sheer simplicity. Getting objects from dropwizard and simply dropping them into a mongo collection is dead simple. Of course, I could have used as well [morphia](http://code.google.com/p/morphia/).

## Setting up MongoDB

I will not tell about any mongo specific stuff here, install MongoDB get it up and running is a task of minutes. So just do it before. In case you are wondering: I do like the [JDBI approach](http://dropwizard.codahale.com/manual/db/) of dropwizard very much as I dont like the idea of ORMs with SQL, but I like the idea of ORMs with data that already is an object, like BSON based data - this always wins. In case you are bound to SQL, JDBI looks like a good thing to me - similar to [ANORM](http://www.playframework.org/documentation/2.0/ScalaAnorm) of the playframework scala API.

## Setting up the project

Create a maven project with maven pom.xml like this

{% highlight xml %}
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>de.spinscale</groupId>
    <artifactId>checkout-api</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>A first dropwizard test project</name>
    <description>A first dropwizard test project</description>
    <dependencies>
        <dependency>
            <groupId>com.yammer.dropwizard</groupId>
            <artifactId>dropwizard-core</artifactId>
            <version>0.3.1</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>net.vz.mongodb.jackson</groupId>
            <artifactId>mongo-jackson-mapper</artifactId>
            <version>1.4.1</version>
        </dependency>        
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>1.4</version>
                <configuration>
                    <createDependencyReducedPom>true</createDependencyReducedPom>
                    <filters>
                        <filter>
                            <artifact>*:*</artifact>
                            <excludes>
                                <exclude>META-INF/*.SF</exclude>
                                <exclude>META-INF/*.DSA</exclude>
                                <exclude>META-INF/*.RSA</exclude>
                            </excludes>
                        </filter>
                    </filters>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer" />
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>service.ProductService</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
{% endhighlight %}

## Define some models

First we need to define some models. In this case we will have a representation of a product - which consists of many variants. Think of a T-Shirt which is available in different colors and sizes. Each variant is a concrete physical representation of a product like a blue shirt in XL. The `lucid` field in the `ProductData` entity can be seen as a unique hash value of your product, some sort of internal identifier, where as the SKU can be seen as the external identifier.

In case you are missing getters and setters - I did not use them in this example to keep the code short. As soon as you want to execute some logic in them, you should of course add them as it is only a 5 second IDE step.

{% highlight java %}
package model;

import java.util.List;
import javax.validation.Valid;
import net.vz.mongodb.jackson.Id;
import net.vz.mongodb.jackson.ObjectId;

public class Product {

    @Id @ObjectId
    public String id;

    @Valid
    public ProductData data;

    @Valid
    public List<ProductData> variants;

}
{% endhighlight %}

Be aware, that I did not use the <code>@Id</code> and <code>@ObjectId</code> annotations from the MongoDB driver but rather the mongo-jackson-driver. The `@Valid` annotation is provided by hibernate validator, which is part of dropwizard.

{% highlight java %}
package model;

import java.util.Map;
import org.hibernate.validator.constraints.NotEmpty;
import com.google.common.collect.Maps;

public class ProductData {

    @NotEmpty
    public String lucid;
    @NotEmpty
    public String sku;
    @NotEmpty
    public String name;

    public Double price;

    public Map<String, String> attributes = Maps.newHashMap();
}
{% endhighlight %}

Again the annotations come from hibernate validator (and are standardized in JSR-303: Bean Validation). As this product data is intended to be used as embedded object, there are not annotations marking a field as ObjectId.

## Add a configuration

The only configuration needed in this example is the connection to MongoDB host. The config file, which is used to start looks like this:

{% highlight yaml %}
mongohost: localhost
mongoport: 27017
mongodb: mydb
{% endhighlight %}

Also you need a java class to read this stuff. As there are default values for everything, the `@NotEmpty` annotation here is rather useless of course. If the value would be empty, your dropwizard service would not start at all due to missing configuration.

{% highlight java %}
package configuration;

import javax.validation.constraints.Max;
import javax.validation.constraints.Min;

import org.codehaus.jackson.annotate.JsonProperty;
import org.hibernate.validator.constraints.NotEmpty;
import com.yammer.dropwizard.config.Configuration;

public class ProductConfiguration extends Configuration {

    @JsonProperty @NotEmpty
    public String mongohost = "localhost";

    @Min(1)
    @Max(65535)
    @JsonProperty
    public int mongoport = 27017;

    @JsonProperty @NotEmpty
    public String mongodb = "yourdb";

}
{% endhighlight %}

## Create a service

The service is the initial glue code tieing your configuration, health checks and resources together.
{% highlight java %}
package service;

import health.MongoHealthCheck;
import model.Product;
import net.vz.mongodb.jackson.JacksonDBCollection;
import resources.product.CreateProductResource;
import resources.product.ProductResource;
import resources.product.RemoveProductVariantResource;

import com.mongodb.DB;
import com.mongodb.Mongo;
import com.yammer.dropwizard.Service;
import com.yammer.dropwizard.config.Environment;

import configuration.ProductConfiguration;

public class ProductService extends Service<ProductConfiguration> {

    public static void main(String[] args) throws Exception {
        new ProductService().run(new String[]{"server", "src/main/resources/config.yaml"});
    }

    private ProductService() {
        super("app");
    }

    @Override
    protected void initialize(ProductConfiguration configuration, Environment environment) throws Exception {
        Mongo mongo = new Mongo(configuration.mongohost, configuration.mongoport);
        DB db = mongo.getDB(configuration.mongodb);
        JacksonDBCollection<Product, String> products =
            JacksonDBCollection.wrap(db.getCollection("products"), Product.class, String.class);

        MongoManaged mongoManaged = new MongoManaged(mongo);
        environment.manage(mongoManaged);

        environment.addHealthCheck(new MongoHealthCheck(mongo));

        environment.addResource(new CreateProductResource(products));
        environment.addResource(new ProductResource(products));
        environment.addResource(new RemoveProductVariantResource(products));
    }
}
{% endhighlight %}

So what has happened here? The `initialize()` method is run on startup by dropwizard. A new `Mongo` object is created from the configuration as well as a `DB` object. The next step is to wrap a normal mongo collection into a jackson db collection, which is then handed over to all resources, and used for querying and inserting objects from then on.

The addition of the health check and the managed instance are explained below.

The created main method also is a helper to start the application easily in your IDE - easy to run integration tests against it from then on.

## Create a managed mongo instance

Managed instances are simple interfaces with a `start()` and `stop()` method allowing you to close open connections. Most simple code has applied. In case you are asking why I left the `start()` method empty.

{% highlight java %}
package service;

import com.mongodb.Mongo;
import com.yammer.dropwizard.lifecycle.Managed;

public class MongoManaged implements Managed {

    private Mongo m;

    public MongoManaged(Mongo m) {
        this.m = m;
    }

    public void start() throws Exception {
    }

    public void stop() throws Exception {
        m.close();
    }
}
{% endhighlight %}


## Create a health check

Health checks are one of the coolest features of dropwizard and show that is has been developed with operations folks in mind. You can develop simple checks to ensure the inner livings of your applications are up and running instead of running a simple icinga check on your application.

This health check simply checks for a working mongo db connection by trying to get all database names. If it fails a mongo exception is thrown and `Result.healthy()` is never returned.

{% highlight java %}
package health;

import com.mongodb.Mongo;
import com.yammer.metrics.core.HealthCheck;

public class MongoHealthCheck extends HealthCheck {

    private Mongo mongo;

    public MongoHealthCheck(Mongo mongo) {
        super("MongoHealthCheck");
        this.mongo = mongo;
    }

    @Override
    protected Result check() throws Exception {
        mongo.getDatabaseNames();
        return Result.healthy();
    }

}
{% endhighlight %}


## Create some helpers

Before taking a look at the resources, I would want to create some helpers, which make the core more readable. They are primarly stolen from the play framework, which does this since ever. The most requests work like this: Get some id from the request, get the object with the id from the persistence store, do something with it, create a response. If no object with the id from the request can be found, return immediately with an error message. The standard error message defined for this in HTTP is 404. So a `notFoundIfNull(object)` method is a very convenient helper. The same also applies if a result of a DBCursor query has no entry at all.

{% highlight java %}
package utils;

import javax.ws.rs.WebApplicationException;
import javax.ws.rs.core.Response.Status;

import net.vz.mongodb.jackson.DBCursor;

public class ResourceHelper {

    public static void notFoundIfNull(DBCursor<?> cursor) {
        if (!cursor.hasNext()) {
            throw new WebApplicationException(Status.NOT_FOUND);
        }
    }

    public static void notFoundIfNull(Object obj) {
        errorIfNull(obj, Status.NOT_FOUND);
    }

    public static void errorIfNull(Object obj, Status status) {
        if (obj == null) {
            throw new WebApplicationException(status);
        }
    }
}
{% endhighlight %}

## Add resources

After all this time of preparation, the time has come to really take a look what happens, when we try to store or read data to/from MongoDB. Lets start with inserting data:

{% highlight java %}
package resources.product;

import javax.validation.Valid;
import javax.ws.rs.PUT;
import javax.ws.rs.Path;
import javax.ws.rs.core.Response;
import javax.ws.rs.core.Response.Status;

import model.Product;
import net.vz.mongodb.jackson.DBCursor;
import net.vz.mongodb.jackson.JacksonDBCollection;

@Path("/product/")
public class CreateProductResource {

    private JacksonDBCollection<Product, String> products;

    public CreateProductResource(JacksonDBCollection<Product, String> products) {
        this.products = products;
    }

    @PUT
    public Response createProduct(@Valid Product product) {
        DBCursor<Product> cursor = products.find().is("data.lucid", product.data.lucid);
        if (cursor.hasNext()) {
            return Response.status(Status.CONFLICT).build();
        }

        products.save(product);

        return Response.noContent().build();
    }
}
{% endhighlight %}

A PUT call to the `/products` URL will add a product only in case the lucid has not already been stored, otherwise a HTTP 409 error message will be returned. If storing was successful a HTTP 204 will be returned, indicating that no additional content will be sent in the body. Of course, in order to be more resty, the URL of the newly created entity should be returned here - like `/products/$lucid`.

Be aware that one resource can trigger several logic exections, as calling a HTTP PUT or HTTP DELETE on the same URL triggers different actions, like in the following resource, which allows to retrieve product data and to delete a complete product.

{% highlight java %}
package resources.product;

import static utils.ResourceHelper.*;
import static javax.ws.rs.*;

import javax.ws.rs.Path;
import javax.ws.rs.PathParam;
import javax.ws.rs.core.Response;
import model.Product;
import net.vz.mongodb.jackson.DBCursor;
import net.vz.mongodb.jackson.JacksonDBCollection;
import net.vz.mongodb.jackson.WriteResult;
import com.mongodb.BasicDBObject;
import com.mongodb.DBObject;

@Path("/product/{id}")
public class ProductResource {

    private JacksonDBCollection<Product, String> products;

    public ProductResource(JacksonDBCollection<Product, String> products) {
        this.products = products;
    }

    @GET
    public Product getProduct(@PathParam("id") String id) {
        DBCursor<Product> cursor = products.find().is("data.lucid", id);
        notFoundIfNull(cursor);

        return cursor.next();
    }

    @DELETE
    public Response removeProduct(@PathParam("id") String id) {
        DBObject query = new BasicDBObject("data.lucid", id);
        WriteResult<Product,String> result = products.remove(query);
        int affectedObjects = result.getWriteResult().getN();

        if (affectedObjects != 1) {
            return Response.serverError().build();
        }
        return Response.noContent().build();
    }
}
{% endhighlight %}

This resource uses the `notFoundIfNull()` method for a db cursor. The `id` parameter is stripped from the request URI as you can see due to the use of the `@Path` and `@PathParam` annotations. The `removeProducts()` method is a little bit hacky and I am pretty sure there is a better way it. It does a delete-by-object, which matches the lucid. In case the count of affected object is not one, something must have gone wrong. Also, here you would want to check on way more error cases as of only this one.

A last simple functionality is the removal of a specific variant. Imagine the red shirt in M went out of stock, so it needs to be removed from mongodb which might be used to display the products in the store.

{% highlight java %}
package resources.product;

import static com.google.common.base.Predicates.not;
import static utils.ResourceHelper.notFoundIfNull;

import java.util.List;

import javax.ws.rs.DELETE;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;
import javax.ws.rs.core.Response;

import model.Product;
import model.ProductData;
import net.vz.mongodb.jackson.DBCursor;
import net.vz.mongodb.jackson.JacksonDBCollection;

import com.google.common.base.Predicate;
import com.google.common.collect.Iterables;
import com.google.common.collect.Lists;

@Path("/product/{id}/{variant}")
public class RemoveProductVariantResource {

    private JacksonDBCollection<Product, String> products;

    public RemoveProductVariantResource(JacksonDBCollection<Product, String> products) {
        this.products = products;
    }

    @DELETE
    public Response removeVariant(@PathParam("id") String id, @PathParam("variant") String variant) {
        DBCursor<Product> cursor = products.find().in("data.lucid", id);
        notFoundIfNull(cursor);

        Product product = cursor.next();

        int size = product.variants.size();
        product.variants = removeVariantByLucid(product, variant);
        if (product.variants.size() == size) {
            return Response.notModified().build();
        }

        products.save(product);

        return Response.noContent().build();
    }

    public List<ProductData> removeVariantByLucid(Product product, String lucid) {
        Iterable<ProductData> iterables = Iterables.filter(product.variants,
            not(new MatchesLucidPredicate(lucid)));
        return Lists.newArrayList(iterables);
    }

    private static final class MatchesLucidPredicate implements Predicate<ProductData> {
        private String lucid;

        public MatchesLucidPredicate(String lucid) {
            this.lucid = lucid;
        }
        public boolean apply(ProductData input) {
            return lucid.equals(input.lucid);
        }
    }
}
{% endhighlight %}

What happens here is that the main product, which contains all the variants is looked up by lucids. After that the correct variant with the corresponding lucid is tried to be found and removed. I am using Google Guava predicates for this - one of the most awesome libraries in the java world. If you havent checked it out yet, you definately should. In case no removal has happened, which can simply be checked if the size of the two lists before and after removal are the same size, a HTTP 304 Not-Modified response is returned. If a removal has happened, the product without the variant is stored and a HTTP 204 response is returned.

Again, I am sure, this deletion is easier to achieve in MongoDB itself via a query, but I have not yet had the time to look it up more exactly.

## Test it using curl

For the sake of lazyness I will be using curl instead of wonderfully written component tests to show the functionality. You can start via the `main()` method in the checkout inside of your IDE.

The first step is to create a new product.

    curl -X PUT localhost:8080/product/ -v --header "Content-Type: application/json" -d' {
    "data" : {
        "sku" : "sku123",
        "lucid": "lucid123",
        "name": "T-Shirt foo"
    },
    "variants" : [
        {
            "sku" : "sku1",
            "lucid": "lucid1",
            "name": "Great variant product",
            "price" : 20.0,
            "attributes" : { "color": "yellow", "material" : "wool", "size": "L" }
        },
        {
            "sku" : "sku2",
            "lucid": "lucid2",
            "name": "Great variant product 2",
            "price" : 30.0,
            "attributes" : { "color": "green", "material" : "wool", "size": "M" }
        }
    ]
}'

Now you can get the created product (and format it nicely via python)

    # curl -s localhost:8080/product/lucid123 --header "Content-Type: application/json" | python -mjson.tool
    {
    "data": {
        "attributes": {}, 
        "lucid": "lucid123", 
        "name": "T-Shirt foo", 
        "price": null, 
        "sku": "sku123"
    }, 
    "id": "4f6f8e640364df7bca4018ca", 
    "variants": [
        {
            "attributes": {
                "color": "yellow", 
                "material": "wool", 
                "size": "L"
            }, 
            "lucid": "lucid1", 
            "name": "Great variant product", 
            "price": 20.0, 
            "sku": "sku1"
        }, 
        {
            "attributes": {
                "color": "green", 
                "material": "wool", 
                "size": "M"
            }, 
            "lucid": "lucid2", 
            "name": "Great variant product 2", 
            "price": 30.0, 
            "sku": "sku2"
        }
    ]
    }

You can delete the second variant by calling

    curl -X DELETE localhost:8080/product/lucid123/lucid2 -v 

Last but not least you can delete the whole product again

    curl -X DELETE localhost:8080/product/lucid123 -v


### Testing the health check

You can go to [localhost:8081/health](http://localhost:8081/healthcheck) and see the healthcheck in action. 

    * MongoHealthCheck: OK
    * deadlocks: OK

Stop MongoDB and reload the page and you will see an exception.

    ! MongoHealthCheck: ERROR

As you can see, working health checks begins with a `*` where as broken services start with a `!` - easy to parse for monitoring.

And thats the little introduction howto for now...

## Todo

* Create a better mongo configuration file and reader. This is easily possible by creating an own MongoConfiguration class. [More about this here](http://dropwizard.codahale.com/manual/core/#configuration). The file should look like this:
{% highlight yaml %}
mongo:
  host: localhost
  port: 27017
  db: mydb
{% endhighlight %}
* Authentication is missing. But this seems super simple, check [it out here](http://dropwizard.codahale.com/manual/auth/)

## And there is more

* [Tasks](http://dropwizard.codahale.com/manual/core/#tasks) are single actions which can be triggered via HTTP
* [Scala support](http://dropwizard.codahale.com/manual/scala/)
* [Templating via Freemarker](http://dropwizard.codahale.com/manual/views/)
* [Integration with databases](http://dropwizard.codahale.com/manual/db/) via JDBI
* Monitoring &amp; [Metrics](http://dropwizard.codahale.com/manual/core/#metrics), which can be checked via [port 8081](http://localhost:8081/metrics?pretty=true)

## What are the cons?

The cons are merely open questions, which need some more investigations and should not be seen as limitations of dropwizard but rather my expierence. Feel to enlighten me :-)

* I have not yet investigated how to provide a nice XML API for my services. But Jackson has a [nice solution for that possibly](https://github.com/FasterXML/jackson-module-jaxb-annotations)
* I would love to have live code reloading/hot swapping. Really. A lot. This would be the killer feature for this framework. Seriously. I would be sold then.
* Freemarker as a templating language. After working for several years with it, I hate it. It is not intuitive for web developers, though it is nice to extend for java developers (and yet again bad to test in frameworks like apache OFBiz)
* I also dislike maven. A bit. While it downloads half the internet. After that always a bit more. But as long as you dont use it for more than dependency management in your projects it is ok.
* I would like to offload the creation of the mongo db connection into the managed instance, but this did not work out, as its start() method seems to be executed to late in order to hand over initialized objects to the resources. This makes the code in the service ugly.

If I wrote something wrong, drop a mail and I will correct it.

