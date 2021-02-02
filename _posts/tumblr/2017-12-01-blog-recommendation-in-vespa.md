---
layout: post
title: Blog recommendation in Vespa
date: '2017-12-01T18:42:26-08:00'
tags: []
tumblr_url: https://blog.vespa.ai/post/168083726636/blog-recommendation-in-vespa
---
## Introduction

This post builds upon the previous [blog search application]({% post_url /tumblr/2017-11-20-blog-search-application-in-vespa %}) and extends the basic search engine to include machine learned models to help us recommend blog posts to users that arrive at our application. Assume that once a user arrives, we obtain his user identification number, denoted in here by `user_id`, and that we will send this information down to Vespa and expect to obtain a blog post recommendation list containing 100 blog posts tailored for that specific user.

Prerequisites:

- Install and build files - code source and build instructions for sbt and Spark is found at [Vespa Tutorial pt. 2](https://github.com/vespa-engine/sample-apps/tree/master/blog-tutorial-shared#vespa-tutorial-pt-2)
- Install [Pig and Hadoop](https://docs.vespa.ai/en/tutorials/blog-recommendation.html#vespa-and-hadoop)
- Put `trainPosts.json` in $VESPA\_SAMPLE\_APPS, the directory with the clone of [vespa sample apps](https://github.com/vespa-engine/sample-apps)
- Put [vespa-hadoop.jar](http://search.maven.org/#search%7Cga%7C1%7Cvespa-hadoop) in $VESPA\_SAMPLE\_APPS
- docker as in the [blog search tutorial](https://docs.vespa.ai/en/tutorials/blog-search.html)

## Collaborative Filtering

We will start our recommendation system by implementing the collaborative filtering algorithm for implicit feedback described in (Hu et. al. 2008). The data is said to be implicit because the users did not explicitly rate each blog post they have read. Instead, the have “liked” blog posts they have likely enjoyed (positive feedback) but did not have the chance to “dislike” blog posts they did not enjoy (absence of negative feedback). Because of that, implicit feedback is said to be inherently noisy and the fact that a user did not “like” a blog post might have many different reasons not related with his negative feelings about that blog post.

In terms of modeling, a big difference between explicit and implicit feedback datasets is that the ratings for the explicit feedback are typically unknown for the majority of user-item pairs and are treated as missing values and ignored by the training algorithm. For an implicit dataset, we would assume a rating of zero in case the user has not liked a blog post. To encode the fact that a value of zero could come from different reasons we will use the concept of confidence as introduced by (Hu et. al. 2008), which causes the positive feedback to have a higher weight than a negative feedback.

Once we train the collaborative filtering model, we will have one vector representing a latent factor for each user and item contained in the training set. Those vectors will later be used in the Vespa ranking framework to make recommendations to a user based on the dot product between the user and documents latent factors. An obvious problem with this approach is that new users and new documents will not have those latent factors available to them. This is what is called a cold start problem and will be addressed with content-based techniques described in future posts.

## Evaluation metrics

The evaluation metric used by [Kaggle](https://www.kaggle.com/c/predict-wordpress-likes/details/evaluation) for this challenge was the Mean Average Precision at 100 (MAP@100). However, since we do not have information about which blog posts the users did not like (that is, we have only positive feedback) and our inability to obtain user behavior to the recommendations we make (this is an offline evaluation, different from the usual A/B testing performed by companies that use recommendation systems), we offer a similar remark as the one included in (Hu et. al. 2008) and prefer recall-oriented measures. Following (Hu et. al. 2008) we will use the expected percentile ranking.

## Evaluation Framework

**Generate training and test sets**

In order to evaluate the gains obtained by the recommendation system when we start to improve it with more accurate algorithms, we will split the dataset we have available into training and test sets. The training set will contain document (blog post) and user action (likes) pairs as well as any information available about the documents contained in the training set. There is no additional information about the users besides the blog posts they have liked. The test set will be formed by a series of documents available to be recommended and a set of users to whom we need to make recommendations. This list of test set documents constitutes the Vespa content pool, which is the set of documents stored in Vespa that are available to be served to users. The user actions will be hidden from the test set and used later to evaluate the recommendations made by Vespa.

To create an application that more closely resembles the challenges faced by companies when building their recommendation systems, we decided to construct the training and test sets in such a way that:

- There will be blog posts that had been liked in the training set by a set of users and that had also been liked in the test set by another set of users, even though this information will be hidden in the test set. Those cases are interesting to evaluate if the exploitation (as opposed to exploration) component of the system is working well. That is, if we are able to identify high quality blog posts based on the available information during training and exploit this knowledge by recommending those high quality blog posts to another set of users that might like them as well.
- There will be blog posts in the test set that had never been seen in the training set. Those cases are interesting in order to evaluate how the system deals with the cold-start problem. Systems that are too biased towards exploitation will fail to recommend new and unexplored blog posts, leading to a feedback loop that will cause the system to focus into a small share of the available content.

A key challenge faced by recommender system designers is how to balance the exploitation/exploration components of their system, and our training/test set split outlined above will try to replicate this challenge in our application. Notice that this split is different from the approach taken by the [Kaggle competition](https://www.kaggle.com/c/predict-wordpress-likes) where the blog posts available in the test set had never been seen in the training set, which removes the exploitation component of the equation.

The Spark job uses `trainPosts.json` and creates the folders `blog-job/training_set_ids` and `blog-job/test_set_ids` containing files with `post_id` and `user_id`pairs:

    $ cd blog-recommendation; export SPARK_LOCAL_IP="127.0.0.1"
    $ spark-submit --class "com.yahoo.example.blog.BlogRecommendationApp" \
      --master local[4] ../blog-tutorial-shared/target/scala-*/blog-support*.jar \
      --task split_set --input_file ../trainPosts.json \
      --test_perc_stage1 0.05 --test_perc_stage2 0.20 --seed 123 \
      --output_path blog-job/training_and_test_indices

- test\_perc\_stage1: The percentage of the blog posts that will be located only on the test set (exploration component).
- test\_perc\_stage2: The percentage of the remaining (post\_id, user\_id) pairs that should be moved to the test set (exploitation component).
- seed: seed value used in order to replicate results if required.

**Compute user and item latent factors**

Use the complete training set to compute user and item latent factors. We will leave the discussion about tuning and performance improvement of the model used to the section about model tuning and offline evaluation. Submit the [Spark job](https://github.com/vespa-engine/sample-apps/tree/master/blog-tutorial-shared#building-and-running-the-spark-script-for-calculating-latent-factors) to compute the user and item latent factors:

    $ spark-submit --class "com.yahoo.example.blog.BlogRecommendationApp" \
      --master local[4] ../blog-tutorial-shared/target/scala-*/blog-support*.jar \
      --task collaborative_filtering \
      --input_file blog-job/training_and_test_indices/training_set_ids \
      --rank 10 --numIterations 10 --lambda 0.01 \
      --output_path blog-job/user_item_cf

Verify the vectors for the latent factors for users and posts:

    $ head -1 blog-job/user_item_cf/user_features/part-00000 | python -m json.tool
    {
        "user_id": 270,
        "user_item_cf": {
            "user_item_cf:0": -1.750116e-05,
            "user_item_cf:1": 9.730623e-05,
            "user_item_cf:2": 8.515047e-05,
            "user_item_cf:3": 6.9297894e-05,
            "user_item_cf:4": 7.343942e-05,
            "user_item_cf:5": -0.00017635927,
            "user_item_cf:6": 5.7642872e-05,
            "user_item_cf:7": -6.6685796e-05,
            "user_item_cf:8": 8.5506894e-05,
            "user_item_cf:9": -1.7209566e-05
        }
    }
    $ head -1 blog-job/user_item_cf/product_features/part-00000 | python -m json.tool
    {
        "post_id": 20,
        "user_item_cf": {
            "user_item_cf:0": 0.0019320602,
            "user_item_cf:1": -0.004728486,
            "user_item_cf:2": 0.0032499845,
            "user_item_cf:3": -0.006453364,
            "user_item_cf:4": 0.0015929453,
            "user_item_cf:5": -0.00420313,
            "user_item_cf:6": 0.009350027,
            "user_item_cf:7": -0.0015649397,
            "user_item_cf:8": 0.009262732,
            "user_item_cf:9": -0.0030964287
        }
    }

At this point, the vectors with latent factors can be added to posts and users.

## Add vectors to search definitions using tensors

Modern machine learning applications often make use of large, multidimensional feature spaces and perform complex operations on those features, such as in large logistic regression and deep learning models. It is therefore necessary to have an expressive framework to define and evaluate ranking expressions of such complexity at scale.

Vespa comes with a Tensor framework, which unify and generalizes scalar, vector and matrix operations, handles the sparseness inherent to most machine learning application (most cases evaluated by the model is lacking values for most of the features) and allow for models to be continuously updated. Additional information about the Tensor framework can be found in the [tensor user guide](https://docs.vespa.ai/en/tensor-user-guide.html).

We want to have those latent factors available in a Tensor representation to be used during ranking by the Tensor framework. A tensor field `user_item_cf` is added to `blog_post.sd` to hold the blog post latent factor:

    field user_item_cf type tensor(user_item_cf[10]) {
    	indexing: summary | attribute
    	attribute: tensor(user_item_cf[10])
    }
    
    field has_user_item_cf type byte {
    	indexing: summary | attribute
    	attribute: fast-search
    }

A new search definition `user.sd` defines a document type named `user` to hold information for users:

    search user {
        document user {
            field user_id type string {
                indexing: summary | attribute
                attribute: fast-search
            }
    
            field has_read_items type array<string> {
                indexing: summary | attribute
            }
    
            field user_item_cf type tensor(user_item_cf[10]) {
                indexing: summary | attribute
                attribute: tensor(user_item_cf[10])
            }
    
            field has_user_item_cf type byte {
                indexing: summary | attribute
                attribute: fast-search
            }
        }
    }

Where:

- user\_id: unique identifier for the user
- user\_item\_cf: tensor that will hold the user latent factor
- has\_user\_item\_cf: flag to indicate the user has a latent factor

## Join and feed data

Build and deploy the application:

    $ mvn install

Deploy the application (in the Docker container):

    $ vespa-deploy prepare /vespa-sample-apps/blog-recommendation/target/application && \
      vespa-deploy activate

Wait for app to activate (200 OK):

    $ curl -s --head http://localhost:8080/ApplicationStatus

The code to join the latent factors in `blog-job/user_item_cf` into blog\_post and user documents is implemented in [tutorial\_feed\_content\_and\_tensor\_vespa.pig](https://github.com/vespa-engine/sample-apps/blob/master/blog-tutorial-shared/src/main/pig/tutorial_feed_content_and_tensor_vespa.pig). After joining in the new fields, a Vespa feed is generated and fed to Vespa directly from Pig :

    $ pig -Dvespa.feed.defaultport=8080 -Dvespa.feed.random.startup.sleep.ms=0 \
      -x local \
      -f ../blog-tutorial-shared/src/main/pig/tutorial_feed_content_and_tensor_vespa.pig \
      -param VESPA_HADOOP_JAR=../vespa-hadoop*.jar \
      -param DATA_PATH=../trainPosts.json \
      -param TEST_INDICES=blog-job/training_and_test_indices/testing_set_ids \
      -param BLOG_POST_FACTORS=blog-job/user_item_cf/product_features \
      -param USER_FACTORS=blog-job/user_item_cf/user_features \
      -param ENDPOINT=localhost

A successful data join and feed will output:

    Input(s):
    Successfully read 1196111 records from: "file:///Users/kraune/github/vespa-engine/sample-apps/trainPosts.json"
    Successfully read 341416 records from: "file:///Users/kraune/github/vespa-engine/sample-apps/blog-recommendation/blog-job/training_and_test_indices/testing_set_ids"
    Successfully read 323727 records from: "file:///Users/kraune/github/vespa-engine/sample-apps/blog-recommendation/blog-job/user_item_cf/product_features"
    Successfully read 6290 records from: "file:///Users/kraune/github/vespa-engine/sample-apps/blog-recommendation/blog-job/user_item_cf/user_features"
    
    Output(s):
    Successfully stored 286237 records in: "localhost"

Sample blog post and user:

- [localhost:8080/document/v1/blog-recommendation/user/docid/22702951](http://localhost:8080/document/v1/blog-recommendation/user/docid/22702951)
- [localhost:8080/document/v1/blog-recommendation/blog\_post/docid/1838008](http://localhost:8080/document/v1/blog-recommendation/blog_post/docid/1838008)

## Ranking

Set up a rank function to return the best matching blog posts given some user latent factor. Rank the documents using a dot product between the user and blog post latent factors, i.e. the query tensor and blog post tensor dot product (sum of the product of the two tensors) - from `blog_post.sd`:

    rank-profile tensor {
        first-phase {
            expression {
                sum(query(user_item_cf) * attribute(user_item_cf))
            }
        }
    }

Configure the ranking framework to expect that `query(user_item_cf)` is a tensor, and that it is compatible with the attribute in a [query profile type](https://docs.vespa.ai/en/query-profiles.html#query-profile-types) - see `search/query-profiles/types/root.xml` and `search/query-profiles/default.xml`:

    <query-profile-type id="root" inherits="native">
        <field name="ranking.features.query(user_item_cf)" type="tensor(user_item_cf[10])" />
    </query-profile-type>
    
    <query-profile id="default" type="root" />

This configures a ranking feature named `query(user_item_cf)` with type `tensor(user_item_cf[10])`, which defines it as an indexed tensor with 10 elements. This is the same as the attribute, hence the dot product can be computed.

## Query Vespa with a tensor

Test recommendations by sending a tensor with latenct factors: [localhost:8080/search/?yql=select%20\*%20from%20sources%20blog\_post%20where%20has\_user\_item\_cf%20=%201;&ranking=tensor&ranking.features.query(user\_item\_cf)=%7B%7Buser\_item\_cf%3A0%7D%3A0.1%2C%7Buser\_item\_cf%3A1%7D%3A0.1%2C%7Buser\_item\_cf%3A2%7D%3A0.1%2C%7Buser\_item\_cf%3A3%7D%3A0.1%2C%7Buser\_item\_cf%3A4%7D%3A0.1%2C%7Buser\_item\_cf%3A5%7D%3A0.1%2C%7Buser\_item\_cf%3A6%7D%3A0.1%2C%7Buser\_item\_cf%3A7%7D%3A0.1%2C%7Buser\_item\_cf%3A8%7D%3A0.1%2C%7Buser\_item\_cf%3A9%7D%3A0.1%7D](http://localhost:8080/search/?yql=select%20*%20from%20sources%20blog_post%20where%20has_user_item_cf%20=%201;&ranking=tensor&ranking.features.query(user_item_cf)=%7B%7Buser_item_cf%3A0%7D%3A0.1%2C%7Buser_item_cf%3A1%7D%3A0.1%2C%7Buser_item_cf%3A2%7D%3A0.1%2C%7Buser_item_cf%3A3%7D%3A0.1%2C%7Buser_item_cf%3A4%7D%3A0.1%2C%7Buser_item_cf%3A5%7D%3A0.1%2C%7Buser_item_cf%3A6%7D%3A0.1%2C%7Buser_item_cf%3A7%7D%3A0.1%2C%7Buser_item_cf%3A8%7D%3A0.1%2C%7Buser_item_cf%3A9%7D%3A0.1%7D)

The query string, decomposed:

- yql=select \* from sources blog\_post where has\_user\_item\_cf = 1 - this selects all documents of type blog\_post which has a latent factor tensor
- restrict=blog\_post - search only in `blog_post` documents
- ranking=tensor - use the rank-profile `tensor` in `blog_post.sd`.
- ranking.features.query(user\_item\_cf) - send the tensor as user\_item\_cf. As this tensor is defined in the query-profile-type, the ranking framework knows its type (i.e. dimensions) and is able to do a dot product with the attribute of same type. The tensor before URL-encoding:

    {
      {user_item_cf:0}:0.1,
      {user_item_cf:1}:0.1,
      {user_item_cf:2}:0.1,
      {user_item_cf:3}:0.1,
      {user_item_cf:4}:0.1,
      {user_item_cf:5}:0.1,
      {user_item_cf:6}:0.1,
      {user_item_cf:7}:0.1,
      {user_item_cf:8}:0.1,
      {user_item_cf:9}:0.1
    }

## Query Vespa with user id

Next step is to query Vespa by user id, look up the user profile for the user, get the tensor from it and recommend documents based on this tensor (like the query in previous section). The user profiles is fed to Vespa in the `user_item_cf` field of the `user` document type.

In short, set up a [searcher](https://docs.vespa.ai/en/searcher-development.html) to retrieve the user profile by user id - then run the query. When the [Vespa Container](https://docs.vespa.ai/en/jdisc/index.html) receives a request, it will create a `Query` representing it and execute a configured list of such Searcher components, called a [search chain](https://docs.vespa.ai/en/chained-components.html). The `query` object contains all the information needed to create a result to the request while the `Result` encapsulates all the data generated from a `Query`. The `Execution` object keeps track of the call state for an execution of the searchers of a search chain:

    package com.yahoo.example;
    
    import com.yahoo.data.access.Inspectable;
    import com.yahoo.data.access.Inspector;
    import com.yahoo.prelude.query.IntItem;
    import com.yahoo.prelude.query.NotItem;
    import com.yahoo.prelude.query.WordItem;
    import com.yahoo.processing.request.CompoundName;
    import com.yahoo.search.Query;
    import com.yahoo.search.Result;
    import com.yahoo.search.Searcher;
    import com.yahoo.search.querytransform.QueryTreeUtil;
    import com.yahoo.search.result.Hit;
    import com.yahoo.search.searchchain.Execution;
    import com.yahoo.search.searchchain.SearchChain;
    import com.yahoo.tensor.Tensor;
    
    import java.util.ArrayList;
    import java.util.Iterator;
    import java.util.List;
    
    public class UserProfileSearcher extends Searcher {
    
        public Result search(Query query, Execution execution) {
    
            // Get tensor and read items from user profile
            Object userIdProperty = query.properties().get("user_id");
            if (userIdProperty != null) {
                Hit userProfile = retrieveUserProfile(userIdProperty.toString(), execution);
                if (userProfile != null) {
                    addUserProfileTensorToQuery(query, userProfile);
    
                    NotItem notItem = new NotItem();
                    notItem.addItem(new IntItem(1, "has_user_item_cf"));
                    for (String item : getReadItems(userProfile.getField("has_read_items"))){
                        notItem.addItem(new WordItem(item, "post_id"));
                    }
                    QueryTreeUtil.andQueryItemWithRoot(query, notItem);
                }
            }
    
            // Restric to search in blog_posts
            query.getModel().setRestrict("blog_post");
    
            // Rank blog posts using tensor rank profile
            if(query.properties().get("ranking") == null) {
                query.properties().set(new CompoundName("ranking"), "tensor");
            }
    
            return execution.search(query);
        }
    
        private Hit retrieveUserProfile(String userId, Execution execution) {
            Query query = new Query();
            query.getModel().setRestrict("user");
            query.getModel().getQueryTree().setRoot(new WordItem(userId, "user_id"));
            query.setHits(1);
    
            SearchChain vespaChain = execution.searchChainRegistry().getComponent("vespa");
            Result result = new Execution(vespaChain, execution.context()).search(query);
    
            execution.fill(result); // This is needed to get the actual summary data
    
            Iterator<Hit> hiterator = result.hits().deepIterator();
            return hiterator.hasNext() ? hiterator.next() : null;
        }
    
        private void addUserProfileTensorToQuery(Query query, Hit userProfile) {
            Object userItemCf = userProfile.getField("user_item_cf");
            if (userItemCf != null) {
                if (userItemCf instanceof Tensor) {
                    query.getRanking().getFeatures().put("query(user_item_cf)", (Tensor)userItemCf);
                }
                else {
                    query.getRanking().getFeatures().put("query(user_item_cf)", Tensor.from(userItemCf.toString()));
                }
            }
        }
    
        private List<String> getReadItems(Object readItems) {
            List<String> items = new ArrayList<>();
            if (readItems instanceof Inspectable) {
                for (Inspector entry : ((Inspectable)readItems).inspect().entries()) {
                    items.add(entry.asString());
                }
            }
            return items;
        }
    }

The searcher is configured in in `services.xml`:

    <chain id='user' inherits='vespa'>
        <searcher bundle='blog-recommendation' id='com.yahoo.example.UserProfileSearcher' />
    </chain>

Deploy, then query a user to get blog recommendations: [localhost:8080/search/?user\_id=34030991&searchChain=user](http://localhost:8080/search/?user_id=34030991&searchChain=user).

To refine recommendations, add query terms: [localhost:8080/search/?user\_id=34030991&searchChain=user&yql=select%20\*%20from%20sources%20blog\_post%20where%20content%20contains%20%22pegasus%22;](http://localhost:8080/search/?user_id=34030991&searchChain=user&yql=select%20*%20from%20sources%20blog_post%20where%20content%20contains%20%22pegasus%22;)

## Model tuning and offline evaluation

We will now optimize the latent factors using the training set instead of manually picking hyperparameter values as was done in Compute user and item latent factors:

    $ spark-submit --class "com.yahoo.example.blog.BlogRecommendationApp" \
      --master local[4] ../blog-tutorial-shared/target/scala-*/blog-support*.jar \
      --task collaborative_filtering_cv \
      --input_file blog-job/training_and_test_indices/training_set_ids \
      --numIterations 10 --output_path blog-job/user_item_cf_cv

Feed the newly computed latent factors to Vespa as before. Note that we need to update the tensor specification in the search definition in case the size of the latent vectors change. We have used size 10 (rank = 10) in the Compute user and item latent factors section but our cross-validation algorithm above tries different values for rank (10, 50, 100).

    $ pig -Dvespa.feed.defaultport=8080 -Dvespa.feed.random.startup.sleep.ms=0 \
      -x local \
      -f ../blog-tutorial-shared/src/main/pig/tutorial_feed_content_and_tensor_vespa.pig \
      -param VESPA_HADOOP_JAR=../vespa-hadoop*.jar \
      -param DATA_PATH=../trainPosts.json \
      -param TEST_INDICES=blog-job/training_and_test_indices/testing_set_ids \
      -param BLOG_POST_FACTORS=blog-job/user_item_cf_cv/product_features \
      -param USER_FACTORS=blog-job/user_item_cf_cv/user_features \
      -param ENDPOINT=localhost

Run the following script that will use Java UDF VespaQuery from the `vespa-hadoop` to query Vespa for a specific number of blog post recommendations for each user\_id in our test set. With the list of recommendation for each user, we can then compute the expected percentile ranking as described in section Evaluation metrics:

    $ pig \
      -x local \
      -f ../blog-tutorial-shared/src/main/pig/tutorial_compute_metric.pig \
      -param VESPA_HADOOP_JAR=../vespa-hadoop*.jar \
      -param TEST_INDICES=blog-job/training_and_test_indices/testing_set_ids \
      -param BLOG_POST_FACTORS=blog-job/user_item_cf_cv/product_features \
      -param USER_FACTORS=blog-job/user_item_cf_cv/user_features \
      -param NUMBER_RECOMMENDATIONS=100 \
      -param RANKING_NAME=tensor \
      -param OUTPUT=blog-job/metric \
      -param ENDPOINT=localhost:8080

At completion, observe:

    Input(s):
    Successfully read 341416 records from: "file:/sample-apps/blog-recommendation/blog-job/training_and_test_indices/testing_set_ids"
    Output(s):
    Successfully stored 5174 records in: "file:/sample-apps/blog-recommendation/blog-job/metric"

In the next post we will improve accuracy using a simple neural network.

## Vespa and Hadoop

Vespa was designed to keep low-latency performance even at Yahoo-like web scale. This means supporting a large number of concurrent requests as well as a very large number of documents. In the previous tutorial we used a data set that was approximately 5Gb. Data sets of this size do not require a distributed file system for data manipulation. However, we assume that most Vespa users would like at some point to scale their applications up. Therefore, this tutorial uses tools such as [Apache Hadoop](https://hadoop.apache.org/),&nbsp;[Apache Pig](https://pig.apache.org/) and [Apache Spark](http://spark.apache.org/). These can be run locally on a laptop, like in this tutorial. In case you would like to use HDFS (Hadoop Distributed File System) for storing the data, it is just a matter of uploading it to HDFS with the following command:

    $ hadoop fs -put trainPosts.json blog-app/trainPosts.json

If you go with this approach, you need to replace the local file paths with the equivalent HDFS file paths in this tutorial.

Vespa has [a set of tools](https://docs.vespa.ai/en/feed-using-hadoop-pig-oozie.html) to facilitate the interaction between Vespa and the Hadoop ecosystem. These can also be used locally. A Pig script example of feeding to Vespa is as simple as:

    REGISTER vespa-hadoop.jar
    
    DEFINE VespaStorage com.yahoo.vespa.hadoop.pig.VespaStorage();
    
    A = LOAD '<path>' [USING <storage>] [AS <schema>];
    
    -- apply any transformations
    
    STORE A INTO '$ENDPOINT' USING VespaStorage();

Use Pig to feed a file into Vespa:

    $ pig -x local -f feed.pig -p ENDPOINT=endpoint-1,endpoint-2

Here, the -x local option is added to specify that this script is run locally, and will not attempt to retrieve scripts and data from HDFS. You need both Pig and Hadoop libraries installed on your machine to run this locally, but you don’t need to install and start a running instance of Hadoop. More examples of feeding to Vespa from Pig is found in [sample apps](https://github.com/vespa-engine/sample-apps/tree/master/blog-tutorial-shared/src/main/pig).

