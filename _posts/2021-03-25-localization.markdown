---
layout: post
title:  "Reducing the App Size Cost of Localization Strings"
date:   2021-03-25 21:45:46 -0400
categories: jekyll update
---

# Reducing the App Size Cost of Localization Strings

## Introduction

The app download size is important to keep small to have a fast onboarding experience for the user, and to reduce the size of the app on disk. By using this trick for an app with a lot of localized strings, it can be reduced by a decent amount.

## Expected Gain

###  Are there enough strings for this to matter?

This trick will reduce the size of localization strings, both compressed and uncompressed, by roughly 50%. The longer the keys are, and the more languages that have localization strings, the more it will reduce it. Typically, it's a good optimization specifically for very large apps, where the strings can take up several megabytes.

## Explanation

Suppose an app has Localization.strings files for English and Spanish, each with values for keys `key1` and `key2`. Here's the English one:
```
"key1" = "some thing";
"key2" = "other thing";
```

and the Spanish one:

```
"key1" = "alguna cosa";
"key2" = "otra cosa";
```

There's some repetition here: `key1` and `key2` have to be included for each language. However, they could just be stored once, in a separate file. Then, each localization file only needs to store its values, _as long as the order of each language's values matches the keys_. So here, it could repackaged in JSON as:

keys.json
```
["key1", "key2"]
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

This new structure, unfortunately, doesn't work with Apple's existing `NSLocalizedString`/`localizedString(forKey key:, value:, table tableName:)` setup. Instead, it needs a new version of these functions. On the plus side, by creating a new pipeline, it makes further customizations easier in the future. Make sure Apple knows which localizations are supported, so it can display it in the App Store, by setting the [CFBundleLocalizations](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundlelocalizations) key. Also, InfoPlist.strings must be left alone, as Apple expects it to be as it is.

Here's a [git repo](https://github.com/michaeleisel/MiniStrings) with an example of this pipeline.

### Pluralization and .stringsdict

Including pluralized strings in a new pipeline is trickier because of the complexities inherent to it. In my experience, the strings that require pluralization are just a fraction of the overall strings, meaning it doesn't matter much. If you do need support though, feel free to file an issue about it.

## Post-install compression

So far, the article has focused on reducing the app's download size. However, the size of that the app's data after installation is also important. Normally, .strings files (or their .json counterparts here) would just sit there uncompressed after installation. This takes up more space on disk than if they were compressed, and also(?) increases the app size that Apple reports in the App Store. To reduce disk space usage, the files can be zipped individually in the bundle. Then, each time the user opens up the app, lazily decompress the file for the language the user actually needs and cache it somewhere in an uncompressed format. There are other compressions options too besides zip, such as the speedy [Zstandard](https://engineering.fb.com/2018/12/19/core-data/zstandard/), which is what Instagram appears to use for their .strings files.

## Conclusion

This trick is one example of a larger theme: find a way to repackage some type of file in an app to be smaller, then replace Apple's pipeline for accessing it with a custom one. For images, for instance, they could all be delivered in a small modern image format like AVIF (with just a slight loss of quality), then have .png files downloaded later.
