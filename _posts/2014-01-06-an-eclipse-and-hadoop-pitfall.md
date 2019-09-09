---
layout: post
title: An Eclipse and Hadoop Pitfall
date: 2014-01-06 04:01
author: matejoseph
comments: true
categories: [Eclipse, Hadoop, MapReduce, Rant]
---
For one of my course projects I inherited a project that was implementing a dimensionality reduction technique in map reduce. The previous owner of the project exclusively ran it using eclipse and used <a href="http://www.amazon.com/gp/product/1935182684/ref=as_li_ss_tl?ie=UTF8&amp;camp=1789&amp;creative=390957&amp;creativeASIN=1935182684&amp;linkCode=as2&amp;tag=josmat0a-20">Mahout in Action</a><img class="vynbimwtqvcxdqgnwrdl" style="border:none !important;margin:0!important;" alt="" src="http://ir-na.amazon-adsystem.com/e/ir?t=josmat0a-20&amp;l=as2&amp;o=1&amp;a=1935182684" width="1" height="1" border="0" />'s examples as a starting point.
<h2>The Problem I Inherited</h2>
The following pattern was littered through most of the Hadoop jobs:

```java
public class Main extends AbstractJob {

  /**
   * Some parameter to be used by the Mapper and
   * Reducer.
   */
  private static int parameter;

  public int run( String[] args ) throws Exception {
    parameter = 100;
    Job brokenJob = new Job( conf, 'This doesn't work!' );
    brokenJob.setMapperClass( BrokenMapper.class );
    brokenJob.setReducerClass( BrokenReducer.class );
    brokenJob.setInputFormatClass( TextInputFormat.class );
    brokenJob.setOutputFormatClass( TextOutputFormat.class );
    brokenJob.setOutputKeyClass( IntWritable.class);
    brokenJob.setOutputValueClass( IntWritable.class );
    brokenJob.setMapOutputKeyClass( IntWritable.class );
    brokenJob.setMapOutputValueClass( IntWritable.class );
  }

  public static void main( String[] args ) throws Exception {
    ToolRunner.run( new Main(), args );
  }

  /**
   * This mapper is broken because it makes reference to a static
   * variable set by the JobClient, on a different JVM.
   */
  public static class BrokenMapper
    extends Mapper < LongWritable, Text, IntWritable, IntWritable > {

    protected void map( LongWritable key, Text value, Context context )
        throws IOException, InterruptedException {
      /**
       * Do something that makes use of static variable 'paramater'
       */
    }
  }

  /**
   * This reducer has the same problem as the above mapper.
   */
  public static class BrokenReducer
    extends Reducer< IntWritable, IntWritable, IntWritable, IntWritable > {

    public void reduce( IntWritable key, Iterable< VectorWritable > values, Context context )
        throws IOException, InterruptedException {
      /**
       * Do something that makes use of static variable 'paramater'
       */
    }
  }
}
```

The above code works when you run it from Eclipse. It works because using Maven, Eclipse has all of the jars that it needs in order to run. However, it does not run it using hadoop -libjar. It's running it using java -classpath. As a result, everything defaults to the LocalJobRunner. The significance of this is that everything is running on a <strong>single</strong> JVM. As a result, the variables set by the JobClient were visible to the Mapper and Reducer. In a real Hadoop job, this is not the case. The JobClient, Mapper, and Reducers run on separate JVMs, and most likely different machines. When you run the code above, you'll get NullPointerExceptions or weird behaviour resulting from the default values of java primitives. This results in developers unfamiliar with the MapReduce paradigm to make invalid assumptions.
<h2>How I Fixed It</h2>
One easy fix is to pass variables through the job configuration. Then in the setup of the Mapper and Reducer, parse the configuration. The code is given below. I have omitted a lot of code from the above example that would be redundant.

```java
public static class FixedMapper
  extends Mapper < LongWritable, Text, IntWritable, IntWritable > {

  private int parameter;

  protected void setup(Context context)
      throws IOException, InterruptedException {
    Configuration conf = context.getConfiguration();
    // we do not use Confugration.getInt(String) because we want
    // to fail if the value is not present.
    parameter = Integer.parseInt(conf.get('parameter'));
  }

  protected void map( LongWritable key, Text value, Context context )
      throws IOException, InterruptedException {
    /**
     * Do something that makes use of static variable 'paramater'
     */
  }
}
```

Unfortunately, this was not enough. Some of the variables passed between the mappers and reducers were matrices. They were on the order of 100s of megabytes in size. It would be a bad idea to place something so large in Hadoop job's configuration. As a result, I saved it to the distributed cache, and loaded it in the mapper's and reducer's setup method.
<h2>Conclusion</h2>
When you're using Eclipse, understand what it is hiding. In this case, Eclipse was hiding that it uses java -classpath to run the main class from Maven. Secondly, it was also hidden that running the main class using java instead of hadoop uses the LocalJobRunner by default. By doing so, everything runs in a single JVM and you will fail to observe where you're exploiting that false assumption.

Additionally, understand the framework you're using is valuable is valuable in avoiding similar pitfalls. In this case, there was a failure in understanding the MapReduce paradigm. The JobClient cannot shared variables with Mapper and Reducer because they run on separate JVMs. This is not an argument against Eclipse. This merely a warning. I use Eclipse for all of my Java development. However, I have compiled and run java programs from the command line. This pattern extends to any tool you're using. For example, what are Ant and Maven hiding from you to make your life easier?
