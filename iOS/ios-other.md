# iOS其他

```objectivec
- (BOOL)isVPNOn {
    BOOL flag = NO;
    NSString *version = [UIDevice currentDevice].systemVersion;
    // need two ways to judge this.
    if (version.doubleValue  >= 9.0){
      NSDictionary *dict = CFBridgingRelease(CFNetworkCopySystemProxySettings());
      NSArray *keys = [dict[@"__SCOPED__"] allKeys];
      for (NSString *key in keys)  {
        if ([key rangeOfString:@"tap"].location != NSNotFound ||
            [key rangeOfString:@"tun"].location != NSNotFound ||
            [key rangeOfString:@"ipsec"].location != NSNotFound ||
            [key rangeOfString:@"ppp"].location != NSNotFound){
            flag = YES;
            break;
         }
      }
    } else {
      struct ifaddrs *interfaces = NULL;
      struct ifaddrs *temp_addr = NULL;
      int success = 0;
      // retrieve the current interfaces - returns 0 on success
      success = getifaddrs(&interfaces);
      if (success == 0) {
        //Loop through linked list of interfaces
        temp_addr = interfaces;
        while (temp_addr != NULL) {
          NSString *string = [NSString stringWithFormat:@"%s", temp_addr->ifa_name];
          if ([string rangeOfString:@"tap"].location != NSNotFound ||
              [string rangeOfString:@"tun"].location != NSNotFound ||
              [string rangeOfString:@"ipsec"].location != NSNotFound ||
              [string rangeOfString:@"ppp"].location != NSNotFound) {
              flag = YES;
              break;
          }
          temp_addr = temp_addr->ifa_next;
       }
    }
    // Free memory
    freeifaddrs(interfaces);
    return flag;
}
```





```objective-c
## iOS判断是否使用代理IP
#import "CETCProxyStatus.h"

+ (BOOL)getProxyStatus {
    NSDictionary *proxySettings = NSMakeCollectable([(NSDictionary *)CFNetworkCopySystemProxySettings() autorelease]);
    NSArray *proxies = NSMakeCollectable([(NSArray *)CFNetworkCopyProxiesForURL((CFURLRef)[NSURL URLWithString:@"http://www.baidu.com"], (CFDictionaryRef)proxySettings) autorelease]);
    NSDictionary *settings = [proxies objectAtIndex:0];
    NSLog(@"host=%@", [settings objectForKey:(NSString *)kCFProxyHostNameKey]);
    NSLog(@"port=%@", [settings objectForKey:(NSString *)kCFProxyPortNumberKey]);
    NSLog(@"type=%@", [settings objectForKey:(NSString *)kCFProxyTypeKey]);
    if([[settings objectForKey:(NSString *)kCFProxyTypeKey]isEqualToString:@"kCFProxyTypeNone"]) {
      //没有设置代理
      return NO;
    } else {
      //设置代理了
      return YES;
    }
}

```
