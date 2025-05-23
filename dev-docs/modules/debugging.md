---
layout: page_v2
page_type: module
title: Module - Debugging
description: Debugging tools to intercept bid requests and mock their response
module_code : debugging
display_name : Debugging
enable_download : true
sidebarType : 1
---

# Debugging module
{: .no_toc}

- TOC
{:toc }

This module allows to "intercept" bids and replace their contents with arbitrary data for the purposes of testing and development.

Bids intercepted in this way are never seen by bid adapters or their backend SSPs, but they are nonetheless injected into the auction as if they originated from them.

{: .pb-alert .pb-alert-warning :}
For convenience, `debugging` configuration is persisted to the browser's session storage, so that you may type `pbjs.setConfig({debugging: ...})` in the console and reload the page to immediately see the effects. This means that you need to remember to **deactivate debugging (or clear session storage) when you are done**.

<a name="example"></a>

## Usage example

The following will intercept all bids for the ad unit with code "test-div", and replace them with mocks that have `cpm: 10`:

```javascript
pbjs.setConfig({
  debugging: {
    enabled: true,
    intercept: [
      {
        when: {
          // intercept bids from bidderA that have adUnitCode === 'test-div'
          adUnitCode: 'test-div',
          bidder: 'bidderA'
        },
        then: {
          // mock their response with sane defaults and `cpm: 10`
          cpm: 10
        }
      },
    ]
  }
});
```

## Intercept rules

`intercept` is a list of objects each containing the following:

{: .table .table-bordered .table-striped }
|Property |Type              |Required? |Description|
|---------+------------------+----------+----------------------------------------------------------------------------------------------|
|`when`   |Function or Object|yes       |[Match rule](#match) - decides which bids should be intercepted by this rule                  |
|`then`   |Function or Object|no        |[Replace rule](#replace) - decides the contents of the bids that are intercepted by this rule |
|`options`|Object            |no        |[Rule options](#options)                                                                      |

Rules are evaluated on each bid in the order they are provided: the first one that has a matching `when` definition takes the bid out of the normal auction flow and replaces it according to its `then` definition.

<a name="match"></a>

### Match rules

The match rule can be provided as a function that takes the bid request as its only argument and returns `true` if the bid should be intercepted, `false` otherwise. The [example above](#example) could be written as:

```javascript
pbjs.setConfig({
    debugging: {
        enabled: true,
        intercept: [
            {
                when: (bidRequest) => bidRequest.adUnitCode === 'test-div',
                then: {
                    cpm: 10
                }
            }
        ]
    }
})
```

Alternatively, the rule can be expressed as an object, and it matches if for each `key`-`value` pair:

- `bidRequest[key] === value`, or
- `value` is a function and `value(bidRequest[key])` is `true`, or
- `value` is a regular expression and it matches `bidRequest[key]`.

To illustrate, these definitions are equivalent:

```javascript
{ 
    when: {
        adUnitCode: 'test-div'
    }
};
{ 
    when: {
        adUnitCode: (code) => code === 'test-div' 
    }
};
{ 
    when: {
        adUnitCode: /^test-div$/
    }
};
```

<a name="replace"></a>

### Replace rules

The replace rule can be provided as a function that takes the bid request as its only argument and returns an object with the desired response properties. The [first example above](#example) could be written as:

```javascript
pbjs.setConfig({
  debugging: {
    enabled: true,
    intercept: [
      {
        when: {
            adUnitCode: 'test-div',
        },
        then: (bidRequest) => ({cpm: 10})
      },
    ]
  }
});
```

Alternatively, the rule can be expressed as an object, and its `key`-`value` pairs will appear in the response as follows:

- if `value` is a function, then `bidResponse[key]` will be set to `value(bidRequest)`;
- otherwise, `bidResponse[key]` will be set to `value`.

To illustrate, the following definitions are equivalent:

```javascript
{
    then: {
        cpm: 10
    }
};
{
    then: {
        cpm: (bidRequest) => 10
    }
}
```

<a name="options"></a>

### Rule options

{: .table .table-bordered .table-striped }
|Property |Type              |Default value |Description                                                                                                             |
|---------+------------------+--------------+------------------------------------------------------------------------------------------------------------------------|
|`delay`  |Number            |0             |Delay (in milliseconds) before intercepted bids are injected into the auction. Can be used to simulate network latency. |

## Examples

### Force a video bid

```javascript
pbjs.setConfig({
  debugging: {
    enabled: true,
    intercept: [
      {
        when: {
          adUnitCode: "video-adunit",
          bidder: "bidderA"
        },
        then: {
          mediaType: "video",
          source: "client",
          currency: "SEK",
          cpm: 50.00,
          creativeId: "11111",
          width: 640,
          height: 360,
          ttl: 300,
          vastXml: "<?xml version=\"1.0\" encoding=\"UTF-8\"?><VAST version=\"2.0\"><Ad id=\"TestAd\"><InLine><AdSystem>Prebid Test</AdSystem><AdTitle>VAST 2.0 Linear Ad</AdTitle><Error><![CDATA[http://myErrorURL/error]]></Error><Impression><![CDATA[]]></Impression><Creatives><Creative id=\"2014\" AdID=\"20150911\" sequence=\"1\"><Linear><Duration>00:00:15</Duration><TrackingEvents><Tracking event=\"creativeView\"><![CDATA[]]></Tracking><Tracking event=\"start\"><![CDATA[]]></Tracking><Tracking event=\"firstQuartile\"><![CDATA[]]></Tracking><Tracking event=\"midpoint\"><![CDATA[]]></Tracking><Tracking event=\"thirdQuartile\"><![CDATA[]]></Tracking><Tracking event=\"complete\"><![CDATA[]]></Tracking></TrackingEvents><VideoClicks><ClickThrough><![CDATA[http://prebid.org/]]></ClickThrough><ClickTracking><![CDATA[]]></ClickTracking></VideoClicks><MediaFiles><MediaFile delivery=\"progressive\" width=\"960\" height=\"540\" type=\"video/mp4\"><![CDATA[https://s3.amazonaws.com/files.prebid.org/creatives/PrebidLogo.mp4]]></MediaFile></MediaFiles></Linear></Creative></Creatives></InLine></Ad></VAST>"
        }
      },
    ]
  }
});
```

### Force a Native Bid

Note: the native response asset IDs and types must match the request.

```javascript
pbjs.setConfig({
  debugging: {
    enabled: true,
    intercept: [
      {
        when: {
          adUnitCode: "test-div",
          bidder: "bidderA"
        },
        then: {
          cpm: 10,
          bidder: "bidderA",
          mediaType: "native",
          source: "client",
          currency: "EUR",
          cpm: 1.00,
          creativeId: "222222",
          native: { ortb: {
            link: { url: "http://example.com" },
            assets: [{
              id: 1,
              title: { text: "Test Native Creative" }
            },{
              id: 2,
              data: {
                type: 2,
                value: "Test Description"
              }
            },{
              id: 3,
              img: {
                type: 3,
                url: "https://files.prebid.org/creatives/prebid300x250.png"
              }
            },{
              id: 4,
              data: {
                type: 1,
                value: "Prebid"
              }
            }]
          }}
        }
      }
    ]
  }
});
```
