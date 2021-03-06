## A sample application using Twitter OAuth and Algolia Search. 

### You'll need a few things to get started on your own remix:
- Twitter app to insert your Twitter Consumer Key & Secret, Twitter Access Key & Secret. 
- Algolia account with your search key and app id

### What is this used for? 
We use Twitter authenticate a user, on successful authentication of a user we kick off a series of actions prior to bringing the user to a success page with a cookie set.

Once we receive success we kick of a series of promises with `fetchAndIndexTweets` that takes the username from the parameters of the successful authentication. This calls `getTweets` starts a call with a promise to Twitter's `GET_USER_TIMELINE` to grab the users last 200 tweets. Once we successfully receive the timeline, we resolve with another promise to `indexTweets` which takes in our Twitter username, our response from Twitter and the Algolia client. We initiate an index specifically for that user named from their username; push the settings via `pushAlgoliaIndexSettings` and then we iterate over the tweets to build and push objects for Algolia. Once this is complete, we render our success page and a user is able to start searching their top tweets.

## How to Remix into your own! 🎏
- Hit that sweet sweet remix link 
- Sign up for a Twitter app account [here](https://apps.twitter.com/app/new) and add your credentials in the .env file
- Sign up for an Algolia account [here](https://www.algolia.com/cc/glitch) and add your credentials in the .env file

et voilà! You can now authorize tweets on your account and the code will create an index in your Algolia account under the name `usertimeline` and you can choose different options from here. 

Tweet at us for what you create, [Algolia Tweets](https://twitter.com/algolia) we'd love to check it out!

## Deep dive into functions 

Users to authenticate with Twitter, and then get that user’s timeline and send it to Algolia to create an index. We can utilize Twitter’s GET_USER_TIMELINE method here and get all the data we need. We do this within our `getTweets` function and call Twitter here with a few params, the screen_name we obtain from the OAuth, the `count` of tweets we want to retrieve and `include_rts` to be false so we can clean up some of the data we are getting back to only show us our original tweets. 

[Source](https://glitch.com/edit/#!/regal-class?path=server/helpers/twitter.js:6:0)

```
    const params = { screen_name: user.username, count: 200, include_rts: false };
    twitterClient.get('statuses/user_timeline', params, function(error, tweets, response) {
      if (error) {
        reject(error);
      } else {
        resolve(tweets);
      }
    });
```
However, since we get a lot of data back and we don’t want to send that all to Algolia, we created an object with only the things we need to rate tweets with the `tweetsToAlgoliaObjects` function that pulls the data we want out of the Twitter response. After we’ve iterated over all the tweets, we will send that timeline data to the Algolia to create an index. 

[Source](https://glitch.com/edit/#!/regal-class?path=server/helpers/algolia.js:36:0)

```
// we will upload and use for the search
function tweetsToAlgoliaObjects(tweets) {
  const algoliaObjects = [];
  // iterate over tweets and build the algolia record
  for (var i = 0; i < tweets.length; i++){
    var tweet = tweets[i];
    var algoliaObject = {
      // use id_str not id (an int), as this int gets truncated in JS
      // the objectID is the key for the algolia record, and mapping
      // tweet id to object ID guarantees only one copy of the tweet in algolia
      objectID: tweet.id_str,
      id: tweet.id_str,
      text: tweet.text,
      created_at: Date.parse(tweet.created_at) / 1000,
      favorite_count: tweet.favorite_count,
      retweet_count: tweet.retweet_count,
      total_count: tweet.retweet_count + tweet.favorite_count,
      url: `https://twitter.com/${tweet.user.username}/status/${tweet.id_str}`
    };
    algoliaObjects.push(algoliaObject);
  }
  return algoliaObjects;
}
```

Now that we have sent the timeline to Algolia, we want to dive into a bit of what is some behind the scenes stuff that makes searching really awesome and making that sweet sweet ranking magic starts to happen.  

One of the cool things about Algolia’s API is it’s not just for basic search, we can do a lot of things with search to querying the results. So after we indexed those records, we want to work with our settings a bit. 

We added a custom ranking for our results, although this works a bit differently with Algolia than it does with other search services in that you don’t need to create a mathematical formula to see different results. Instead we can take existing data you have, in our case we have the Date, Favorite Count, Retweet Count. The traditional way of thinking of sorting would be by date. What if we did something like Retweet Count with a tie breaker of Favorite Count? 

[Source](https://glitch.com/edit/#!/regal-class?path=server/helpers/algolia.js:69:6)
```
index.setSettings({
   // tweets will be ranked by total count with retweets
   // counting more that other interactions, falling back to date
      customRanking: ['desc(total_count)', 'desc(retweet_count)', 'desc(created_at)'],
   // return these attributes for dislaying in search results
      attributesToRetrieve: [
        'text', 'url', 'retweet_count', 'total_count']
});
```

So, this a great start to have our ranking in a custom manner. The next step is making sure we are searching for the right words. What we mean by this is we want to search for text that is relevant to our user. We wouldn’t want to our search for “Denver” to show up for 90% of our data because that’s the geo location. Instead, we would want to focus on the text in the tweet instead. To achieve this, we can add searchableAttributes to our setSettings function.

[Source](https://glitch.com/edit/#!/regal-class?path=server/helpers/algolia.js:64:6)
```
// only the text of the tweet should be searchable
  searchableAttributes: ['text'],
// only highlight results in the text field
  attributesToHighlight: ['text'],
```

We can take this a few steps further to tidy up our search. Let’s look at plural cases of words and typos. 

[Source](https://glitch.com/edit/#!/regal-class?path=server/helpers/algolia.js:75:6)
```
index.set_settings({
  ignorePlurals: ['en', 'fr']
});
```

