# Twift

[![Twitter API v2 badge](https://img.shields.io/endpoint?url=https%3A%2F%2Ftwbadges.glitch.me%2Fbadges%2Fv2)](https://developer.twitter.com/en/docs/twitter-api/early-access)
[![Documentation Coverage](https://github.com/daneden/Twift/blob/badges/.github/badges/coverage.svg)](https://github.com/daneden/Twift/wiki)

Twift is an asynchronous Swift library for the Twitter v2 API.

- [x] No external dependencies
- [x] Fully async
- [x] Full Swift type definitions/wrappers around Twitter's API objects

## Quick Start Guide

New `Twift` instances must be initiated with either OAuth 2.0 user authentication or an App-Only Bearer Token:

```swift
// OAuth 2.0 User Authentication
let oauthUser: OAuth2User = OAUTH2_USER
let userAuthenticatedClient = Twift(.oauth2UserAuth(oauthUser: oauthUser)

// App-Only Bearer Token
let appOnlyClient = Twift(.appOnly(bearerToken: BEARER_TOKEN)
```

You can authenticate users with `Twift.Authentication().authenticateUser()`:

```swift
var client: Twift?

let (oauthUser, error) = await Twift.Authentication().authenticateUser(
  clientId: TWITTER_CLIENT_ID,
  redirectUri: URL(string: TWITTER_CALLBACK_URL)!,
  scope: Set(OAuth2Scope.allCases)
)

if let oauthUser = oauthUser {
  client = Twift(.oauth2UserAuth(oauthUser))
}
```

Once initiated, you can begin calling methods appropriate for the authentication type:

```swift
do {
  // User objects always return id, name, and username properties,
  // but additional properties can be requested by passing a `fields` parameter
  let authenticatedUser = try await userAuthenticatedClient.getMe(fields: [\.profilePhotoUrl, \.description])
  
  // Non-standard properties are optional and require unwrapping
  if let description = authenticatedUser.description {
    print(description)
  }
} catch {
  print(error.localizedDescription)
}
```

Posting Tweets supports text, polls, and media:

```swift
do {
  let textOnlyTweet = MutableTweet(text: "This is a test Tweet")
  try await twitterClient.postTweet(textOnlyTweet)
  
  let poll = try MutablePoll(options: ["Soft g", "Hard g"])
  let tweetWithPoll = MutableTweet(text: "How do you pronounce 'gif'?", poll: poll)
  try await twitterClient.postTweet(tweetWithPoll)
  
  if let mediaData = UIImage(named: "fluffy-cat.jpeg")?.jpegData(compressionQuality: 1.0) {
    let mediaInfo = try await twitterClient.upload(mediaData: mediaData, mimeType: .jpeg)
    try await twitterClient.addAltText(to: mediaInfo.mediaIdString, text: "A fluffy cat")
    let media = MutableMedia(mediaIds: [mediaInfo.mediaIdString])
    let tweetWithMedia = MutableTweet(text: "Here's a nice photo of a cat", media: media)
    try await twitterClient.postTweet(tweetWithMedia)
  }
} catch {
  print(error)
}
```

## Requirements

> To be completed

## Documentation

You can find the full documentation in [this repo's Wiki](https://github.com/daneden/Twift/wiki) (auto-generated by [SwiftDoc](https://github.com/SwiftDoc/swift-doc)). 

## Quick Tips

### Typical Method Return Types
Twift's methods generally return `TwitterAPI[...]` objects containing up to four properties:

- `data`, which contains the main object(s) you requested (e.g. for the `getUser` endpoint, this contains a `User`)
- `includes`, which includes any expansions you request (e.g. for the `getUser` endpoint, you can optionally request an expansion on `pinnedTweetId`; this would result in the `includes` property containing a `Tweet`)
- `meta`, which includes information about pagination (such as next/previous page tokens and result counts)
- `errors`, which includes an array of non-failing errors

All of the methods are throwing, and will throw either a `TwiftError`, indicating a problem related to the Twift library (such as incorrect parameters passed to a method) or a `TwitterAPIError`, indicating a problem sent from Twitter's API as a response to the request.

###  Fields and Expansions

Many of Twift's methods accept two optional parameters: `fields` and `expansions`. These parameters allow you to request additional `fields` (properties) on requested objects, as well as `expansions` on associated objects. For example:

```swift
// Returns the currently-authenticated user
let response = try? await userAuthenticatedClient.getMe(
  // Asks for additional fields: the profile image URL, and the user's description/bio
  fields: [\.profileImageUrl, \.description],
  
  // Asks for expansions on associated fields; in this case, the pinned Tweet ID.
  // This will result in a Tweet on the returned `TwitterAPIDataAndIncludes.includes`
  expansions: [
    // Asks for additional fields on the Tweet: the Tweet's timestamp, and public metrics (likes, retweets, and replies)
    .pinnedTweetId([
      \.createdAt,
      \.publicMetrics
    ])
  ]
)

// The user object
let me = response?.data

// The user's pinned Tweet
let tweet = response?.includes?.tweets.first
```
