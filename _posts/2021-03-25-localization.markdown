---
layout: post
title:  "How to Cut the Size of .strings Files in Half"
date:   2021-03-25 21:45:46 -0400
categories: jekyll update
---

## Introduction

It's important to minimize the app's size, both to reduce its download time as well as reduce the space it takes up on disk. This trick helps with that specifically for .strings files.

## Expected Gain

It reduces the size of localization strings, both compressed and uncompressed, by roughly 50%. The longer the keys are, and the more languages that have localization strings, the more it will reduce it. Typically, it's best for very large apps, where the strings can take up several megabytes.

## Explanation

Suppose an app has Localization.strings files for English and Spanish, each with the same two key-value pairs. Here's the English one:
```
"SomeLongKey1" = "some thing";
"SomeLongKey2" = "other thing";
```

and the Spanish one:

```
"SomeLongKey1" = "alguna cosa";
"SomeLongKey2" = "otra cosa";
```

There's some repetition here: `SomeLongKey1` and `SomeLongKey2` have to be included for each language. However, they could just be stored once, in a separate file. Then, each localization file only needs to store its values and not its keys, _as long as the order of each language's values matches the keys_. So here, it could repackaged in JSON as:

keys.json
```
["SomeLongKey1", "SomeLongKey2"]
```

english.json
```
["some thing", "other thing"]
```

spanish.json
```
["alguna cosa", "otra cosa"]
```

Now the key-value pairs for any language can be reconstructed by zipping the global list of keys with that language's values. Note that the list of keys must include any key that's needed for at least one language. For languages that don't have a value for a key needed by another language, there needs to be a placeholder. E.g., if Spanish didn't have a value for `key2`, its JSON would be `["alguna cosa", null]`

## A New Strings Pipeline

This new structure doesn't work with Apple's existing `NSLocalizedString`/`localizedString(forKey key:, value:, table tableName:)` setup. Instead, it needs a new version of these functions. On the plus side, by creating a new pipeline, it makes further customizations easier in the future. Make sure Apple knows which localizations are supported, so it can display it in the App Store, by setting the [CFBundleLocalizations](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundlelocalizations) key. Also, InfoPlist.strings must be left alone, as Apple expects it to be as it is.

Here's a [git repo](https://github.com/michaeleisel/MiniStrings) with an example of this pipeline.

## Alternative: Using Maps with Compressed Keys

After recently poking around Instagram's app, I saw they have an interesting alternative: they use the original map format of .strings files, but in JSON and with compressed keys. E.g.:

```
"1044mf": "some thing",
"105a9V": "other thing"
```

Then they can either rewrite the keys in the source code, meaning they don't need a custom pipeline, or else have a separate map from long key to short key that the pipeline uses.

Benefits: Although this will result in a slightly larger app size, especially uncompressed, it has some wins. It eliminates the need for placeholder elements and it allows you to have a custom order for each file, theoretically resulting in a better compressed size if you can find a clever enough way to order it.

## Pluralization and .stringsdict

Including pluralized strings in a new pipeline is trickier because of the complexities inherent to it. In my experience, the strings that require pluralization are just a fraction of the overall strings, meaning it doesn't matter much. If you do need support though, feel free to file an issue about it.

## Post-install compression

So far, the article has focused on reducing the app's download size. However, the size of that the app's data after installation is also important. Normally, .strings files (or their .json counterparts here) would just sit there uncompressed after installation. This takes up more space on disk than if they were compressed, and also(?) increases the app size that Apple reports in the App Store. To reduce disk space usage, the files can be zipped individually in the bundle. Then, each time the user opens up the app, lazily decompress the file for the language the user actually needs and cache it somewhere in an uncompressed format. There are other compressions options too besides zip, such as the speedy [Zstandard](https://engineering.fb.com/2018/12/19/core-data/zstandard/), which is what Instagram appears to use for their .strings files.

## Conclusion

This trick is one example of a larger theme: find a way to repackage some type of file in an app to be smaller, then replace Apple's pipeline for accessing it with a custom one. For images, for instance, they could all be delivered in a small modern image format like AVIF (with just a slight loss of quality), then have .png files downloaded later.
