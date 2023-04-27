---
title: "Generating Images in SVG with Supabase Edge Functions and Dart"
datePublished: Tue Apr 25 2023 22:15:45 GMT+0000 (Coordinated Universal Time)
cuid: clgwtt1q2000109l43unid2pw
slug: generating-images-in-svg-with-supabase-edge-functions-and-dart
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1682457477387/60115a41-d536-4ea4-87d5-94cd5317a70a.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1682457493398/caa8fbaa-1161-49f5-afdc-b051387dddbc.png
tags: dart, images, edge-functions, dart-edge

---

Are you looking for a way to create dynamic images for sharing on social media platforms? If so, you might be interested in learning how to generate Open Graph (OG) images in Scalable Vector Graphics (SVG) format using [Supabase Edge Functions](https://supabase.com/docs/guides/functions) with the Dart programming language using [Dart Edge](https://docs.dartedge.dev/platform/supabase).

Supabase Edge Functions are a great way to execute serverless functions at the edge, enabling you to perform complex operations in real-time. Dart is a powerful, object-oriented language that can be used to create web, mobile, and desktop applications.

## Prerequisites

Before we start, make sure that you have the following:

* [Dart SDK](https://dart.dev/get-dart) installed on your machine
    
* A code editor like [Visual Studio Code](https://code.visualstudio.com/)
    
* [Dart Edge](https://docs.dartedge.dev/):
    
    ```dart
    dart pub global activate edge
    ```
    

## The Plan

Our image generator will have the following features:

* Accept text input to be displayed on the image
    
* Allow the user to choose a background pattern and colors for the pattern and text
    
* Generate an SVG image with the specified text and background
    

## Building the Image Generator

First, we need to import these packages that we are going to use in this example:

```dart
import 'dart:convert';
import 'package:edge/edge.dart';
import 'package:supabase_functions/supabase_functions.dart';
import 'package:xml/xml.dart' as xml;
```

### Defining the main function:

Next, you'll need to define the main function, which sets up the SupabaseFunctions instance and handles requests.

In the main function, you define a SupabaseFunctions instance and specify a fetch function that will handle incoming HTTP requests. The fetch function reads query parameters from the request URL, such as the text to be displayed on the image, the image height and length, and the color scheme to use. It then calls the generateOGImage function, passing in these parameters to generate an SVG image.

Finally, the fetch function returns the SVG image as a response with appropriate headers.

```dart
void main() {
    SupabaseFunctions(fetch: (request) async {
        // Get the text parameter from the request
        final text = request.url.queryParameters['text'] ?? 'Hello, world!';
        // Height & length of the final image
        final height = request.url.queryParameters['height'] ?? '630';
        final length = request.url.queryParameters['length'] ?? '1200';
        // Patern to be used (more on this later)
        final pattern = request.url.queryParameters['p'] ?? '';
        // Main, secondary & text colors to be used
        final pcolor = request.url.queryParameters['pcolor'] ?? '#040703';
        final scolor = request.url.queryParameters['scolor'] ?? '#055C13';
        final text_color = request.url.queryParameters['text_color'] ?? '#FFFFFF';
        // Returns the image with the proper headers:
        final svg = generateOGImage(text, int.parse(height), int.parse(length), pattern, pcolor, scolor, text_color);
        final headers = Headers({'Content-Type': 'image/svg+xml', 'Cache-Control': 'public, max-age=3600'});
        return Response(utf8.encode(svg),
        status: 200,
        headers: headers
                       );
    });
}
```

### Where the magic happens the function to generate the image:

The `generateOGImage` function takes in six parameters:

* `text`: The text to display in the center of the image.
    
* `height`: The height of the image.
    
* `length`: The length of the image.
    
* `pattern`: The name of the pattern to use for the background. The available patterns are: `motif`, `surf`, `coil`, `scribble`, `radial`, and `linear`.
    
* `pcolor`: The primary color of the pattern.
    
* `scolor`: The secondary color of the pattern.
    

The function first selects the appropriate pattern based on the parameters passed to it. If the selected pattern does not exist, it defaults to the `linear` pattern. The function then generates an SVG image using the `xml` package. The image consists of a background pattern and the specified text in the center.

```dart
String generateOGImage(String text, int height, int length, String pattern, String pcolor, String scolor, String text_color) {
    String selectedPattern = getPatternFunction(pattern, length, height);
    List<String> pattern_values = ['motif','surf','coil','scribble','radial','linear'];
    final int text_length = text.length;
    //Checks if the pattern actually exists, if not pick the default
    final actual_pattern = pattern_values.contains(pattern) ? pattern : 'linear';
    final svg = xml.XmlBuilder();
    //Check if the patterns uses more fancy elements or set the default one
    if (selectedPattern.isEmpty) {
        selectedPattern = '<rect x="0" y="0" width="$length" height="$height" fill="url(#$actual_pattern)" />';
    }
    svg.processing('xml', 'version="1.0" encoding="UTF-8"');
    svg.element('svg', nest: () {
        // Set the viewBox attribute to ensure the image scales properly
        svg.attribute('viewBox', '0 0 $length $height');
        svg.attribute('xmlns', 'http://www.w3.org/2000/svg');
        svg.attribute('xmlns:xlink', 'http://www.w3.org/1999/xlink');
        //The SVG patterns used: 
        svg.element('defs', nest: () {
            svg.element('pattern', attributes: {
                            'id': 'motif',
                            'width': '40',
                            'height': '40',
                            'fill': '$pcolor',
                            'patternUnits': 'userSpaceOnUse',
                            'patternTransform': 'translate(19 0) scale(1.4) rotate(55) skewX(0) skewY(0)',
            }, nest: () {
                svg.xml('<rect width="100%" height="100%" fill="url(#linear)"/>');
                svg.xml('<path d="M9.39371 2.87681L34.4892 6.75876L16.118 17.3644L9.39371 2.87681Z" opacity="0.3" fill="$pcolor"></path>');
                svg.xml('<path d="M9.39711 22.8738L9.39697 2.87378L16.1171 17.3638V37.3638L9.39711 22.8738Z" opacity="0.3" fill="$scolor"></path>');
                svg.xml('<path d="M34.4871 26.7538L34.4872 6.75391L16.1173 17.3439L16.1172 37.3638L34.4871 26.7538Z" opacity="0.3" fill="#333333"></path>');
            });
            svg.element('linearGradient', attributes: {
                            'id': 'linear',
                            'gradientTransform': 'rotate(214 .5 .5)',
            }, nest: () {
                svg.element('stop', attributes: {
                                'offset': '0.15',
                                'stop-color': '$pcolor',
                            });
                svg.element('stop', attributes: {
                                'offset': '1',
                                'stop-color': '$scolor',
                            });
            });

            svg.element('radialGradient', attributes: {
                            'id': 'radial',
                            'r': '0.75',
                            'cx': '0.5',
                            'cy': '0.5',
            }, nest: () {
                svg.element('stop', attributes: {
                                'offset': '0',
                                'stop-color': '$pcolor',
                            });
                svg.element('stop', attributes: {
                                'offset': '1',
                                'stop-color': '$scolor',
                            });
            });

            svg.element('linearGradient', attributes: {
                            'id': 'scribble',
                            'x1': '50%',
                            'y1': '0%',
                            'x2': '50%',
                            'y2': '100%',
            }, nest: () {
                svg.element('stop', attributes: {
                                'stop-color': '$pcolor',
                                'stop-opacity': '1',
                                'offset': '0%',
                            });
                svg.element('stop', attributes: {
                                'stop-color': '$scolor',
                                'stop-opacity': '1',
                                'offset': '100%',
                            });
            });
            svg.element('linearGradient', attributes: {
                            'id': 'surf',
                            'x1': '50%',
                            'y1': '0%',
                            'x2': '50%',
                            'y2': '100%',
            }, nest: () {
                svg.element('stop', attributes: {
                                'stop-color': '$pcolor',
                                'stop-opacity': '1',
                                'offset': '0%',
                            });
                svg.element('stop', attributes: {
                                'stop-color': '$scolor',
                                'stop-opacity': '1',
                                'offset': '100%',
                            });
            });

        });
        //Applies the selected Pattern
        svg.xml(selectedPattern);
        int font_size = scaleFontSize(text_length, length);
        svg.element('text', nest: () {
            svg.attribute('x', '50%');
            svg.attribute('y', '50%');
            svg.attribute('fill', '$text_color');
            svg.attribute('font-size', '$font_size');
            svg.attribute('font-family', 'Helvetica');
            svg.attribute('font-weight', 'bold');
            svg.attribute('text-anchor', 'middle');
            //You can include the pattern here to help debugging
            svg.text(text);
        });
    });
    return svg.build().toString();
}
```

### Helper functions

Scaling the font is tricky without the common UI flutter elements, but we can scale them to a rough appropriate size:

```dart
// This function calculates the optimal font size for the given text length and image width
int scaleFontSize(int textLength, int imgWidth) {
  double ratio = imgWidth / (textLength * 10);
  int fontSize = (ratio * 18).round();
  return fontSize;
}
```

Checking if using a 'fancy' pattern:

```dart
// This function returns the SVG pattern corresponding to the given pattern name
String getPatternFunction(String pattern, int length, int height) {
  switch (pattern) {
    case 'surf':
      return generateSurfPattern(length, height);
    case 'coil':
      return generateCoilPattern(length, height);
    default:
      return '';
  }
}
```

SVG pattern functions (just for completion, not worth spending too much time here):

```dart
String generateSurfPattern(int length, int height) {
    List<String> transforms = [
                                  'matrix(1,0,0,1,0,35)',
                                  'matrix(1,0,0,1,0,70)',
                                  'matrix(1,0,0,1,0,105)',
                                  'matrix(1,0,0,1,0,140)',
                                  'matrix(1,0,0,1,0,175)',
                                  'matrix(1,0,0,1,0,210)',
                                  'matrix(1,0,0,1,0,245)',
                              ];
    List<double> opacities = [0.05, 0.21, 0.37, 0.53, 0.68, 0.84, 1.0];
    StringBuffer svg = StringBuffer('<g fill="url(#surf)" transform="matrix(1,0,0,1,0,-90.39413452148438)">');
    svg.write('<rect x="0" y="0" width="$length" height="$height" fill="url(#linear)" />');
    for (int i = 0; i < transforms.length; i++) {
        svg.write('<path d="M 0 342.29282328600317 Q ${length * 450 / 2400} 504.6117272258968 ${length * 600 / 2400} 305.7384012795945 ');
        svg.write('Q ${length * 1050 / 2400} 515.5586511543015 ${length * 1200 / 2400} 304.8942027727378 ');
        svg.write('Q ${length * 1650 / 2400} 588.4072802827825 ${length * 1800 / 2400} 305.3095324794213 ');
        svg.write('Q ${length * 2250 / 2400} 543.4292723926144 $length 300.78824070147607 ');
        svg.write('L $length $height L 0 $height L 0 344.59037112525715 Z" ');
        svg.write('transform="${transforms[i]}" ');
        svg.write('opacity="${opacities[i]}"></path>');
    }
    svg.write('</g>');
    return svg.toString();
}

String generateCoilPattern(int length, int height) {
    final int centerX = length ~/ 2;
    final int centerY = height ~/ 2;
    final int strokeWidth = 9;
    final int numCircles = 8;
    final double opacityStep = 0.05;
    final double rotationStep = 360 / numCircles;
    final double radiusStep = (385 - 245) / (numCircles - 1);
    final List<String> circles = [];

    for (int i = 0; i < numCircles; i++) {
        double radius = 385 - (radiusStep * i);
        double rotation = rotationStep * i;
        double opacity = 0.05 + (opacityStep * i);
        double strokeDashArray1 = 2056.0 - (187.0 * i.toDouble());
        double strokeDashArray2 = 2419.0 - (170.0 * i.toDouble());

        circles.add('''
                    <circle
                    r="$radius"
                    cx="$centerX"
                    cy="$centerY"
                    stroke-width="$strokeWidth"
                    stroke-dasharray="$strokeDashArray1 $strokeDashArray2"
                    transform="rotate($rotation, $centerX, $centerY)"
                    opacity="$opacity">
                    </circle>
                    ''');
    }
    return '''
           <g stroke="url(#coil)" fill="#111111" opacity="0.93" stroke-linecap="round">
           <rect x="0" y="0" width="$length" height="$height" fill="url(#linear)"/>
           ${circles.join('\n')}
           </g>
           ''';
}
```

In conclusion, Supabase Edge Functions, coupled with Dart Edge, offer a convenient solution for generating dynamic images in SVG format, which can be used for various purposes, including social media sharing. With the ability to execute serverless functions at the edge, you can perform complex operations in real-time without worrying about infrastructure. Additionally, Dart provides a powerful, object-oriented programming language that can be utilized to create cross-platform applications. We hope this tutorial has provided you with a clear understanding of how to generate SVG images with Supabase Edge Functions and Dart. You can find the full code here:  
[https://github.com/mansueli/dart\_edge\_generate\_image](https://github.com/mansueli/dart_edge_generate_image)

### Example of pictures:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1682460792205/ae849a70-8074-42b4-8e89-ec983238a388.svg align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1682460785761/27015545-aa85-4426-a332-fd0a11baae76.svg align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1682460778352/f96dea63-4da9-4a4b-a7b6-494058ed3c34.svg align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1682460798975/5c2ede1f-9401-4f53-acfe-38233921562f.svg align="center")