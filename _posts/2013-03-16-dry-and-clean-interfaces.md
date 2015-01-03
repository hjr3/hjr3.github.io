---
title: DRY and Clean Interfaces
layout: post
---

The principle of Don't Repeat Yourself (DRY) is more than just grouping common code together. When trying to apply the DRY principle, it is easy to start making a mess of a class interface. I recently had to write some code to generate Flickr image URLs from an API response. I needed to generate two types of URLs: a thumbnail and a normal image. Here is one version of code reuse:

{% highlight objective-c %}
- (NSString *)generateFlickrImageUrl:(NSDictionary *)photo withImageType:(NSString *) imageType
{
    NSString *imageSize;

    if ([imageType isEqualToString:@"thumbnail"]) {
        imageSize = @"t";
    } else {
        imageSize = @"z";
    }

    NSString *farm = [photo valueForKey:@"farm"];
    NSString *server = [photo valueForKey:@"server"];
    NSString *photoId = [photo valueForKey:@"id"];
    NSString *secret = [photo valueForKey:@"secret"];
    NSString *url = [NSString stringWithFormat:@"http://farm%@.staticflickr.com/%@/%@_%@_%@.jpg", farm, server, photoId, secret, imageSize];
    
    return url;
}
{% endhighlight %}

While this does reuse code, it is bad because the code is not accepting of change. If I have to add some other sort of image type, then I have to modify this function. This is a big red flag. When you start using method parameters as an extension of your interface, you may be making the code hard to change. Also, anyone using this class will have to look for the list of available options for the `imageType` parameter. I think it is better to design a easy to understand interface instead.

{% highlight objective-c %}
- (NSString *)generateFlickrImageUrl:(NSDictionary *)photo withImageSize:(NSString *) imageSize
{
    NSString *farm = [photo valueForKey:@"farm"];
    NSString *server = [photo valueForKey:@"server"];
    NSString *photoId = [photo valueForKey:@"id"];
    NSString *secret = [photo valueForKey:@"secret"];
    NSString *url = [NSString stringWithFormat:@"http://farm%@.staticflickr.com/%@/%@_%@_%@.jpg", farm, server, photoId, secret, imageSize];
    
    return url;
}

- (NSString *)generateImageUrlThumbnail:(NSDictionary *)photo
{
    return [self generateFlickrImageUrl:photo withImageSize:@"t"];
}

- (NSString *)generateImageUrl:(NSDictionary *)photo
{
    return [self generateFlickrImageUrl:photo withImageSize:@"z"];
}
{% endhighlight %}

The `generateFlickrImageUrl` is now a protected method of the class while the `generateImageUrl` and `generateImageUrlThumbnail` methods define the public interface.
