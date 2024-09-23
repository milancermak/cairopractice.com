---
title: Building a fully onchain NFT
date: 2024-09-23
author: m
tags: [starknet]
---

Let's build a fully onchain NFT. It will look something like this:

<img src="/assets/img/nouns_glasses.svg" alt="Nouns Glasses" />

To achieve this, there are three standards we need to stitch together - Data URI, Opensea Metadata Standard and SVG. We'll have a look at them individually first.

### Data URI

A [Data URI](https://en.wikipedia.org/wiki/Data_URI_scheme) is a way how to represent "stuff" inline, in a single string that web browsers and libraries understand. In our case, the "stuff" we need to represent is a JSON object, so our Data URI will look something like this:

```json
data:application/json,{"json":"object"}
```

### Opensea Metadata Standard

What shape should the JSON in our Data URI be? EIP-721 does specify an optional metadata extension, but it's rather clunky. A de facto standard followed by many is [this one from Opensea](https://docs.opensea.io/docs/metadata-standards#metadata-structure) so it's the one we're going to use. At minimum then, our JSON will have `name`, `description` and `image` properties. Here's how it will look as a Data URI:

```json
data:application/json,{"name":"Our collection name","description":"Description of the particular item","image":"TODO"}
```

### SVG

Last thing we need to figure out is how to put an image inside a JSON. Enter, Scalable Vector Graphics. [SVG](https://developer.mozilla.org/en-US/docs/Web/SVG) is a XML-based (i.e. textual) format to do 2D graphics. A small green square in SVG looks like this in code:

```xml
<svg width="50" height="50" xmlns="http://www.w3.org/2000/svg">
  <rect x="0" y="0" width="50" height="50" fill="green" />
</svg>
```

And like this when rendered:

<img src="/assets/img/green_square.svg" alt="Green square" />

SVG may look simple, but it is a rather powerful standard (it can do animations!) and it's widely supported. Every modern browser has a full SVG rendering engine.

This `<svg>...</svg>` string is what we want to put as the content of the `image` attribute from the Opensea Metadata Standard. However, the spec says that `image` is an "URL" of the image and not the image itself. Well, we already know how to solve that - we'll encode the SVG in a Data URI! So the small green square will become this:

```text
data:image/svg+xml,<svg width='50' height='50' xmlns='http://www.w3.org/2000/svg'><rect x='0' y='0' width='50' height='50' fill='green' /></svg>
```

We've only replaced the double quotes with single quotes because SVG permits for using single quotes for arguments but JSON requires double quotes to represent strings.

Note there are alternatives to using raw SVG to build onchain NFTs. A great example is [Blobert](https://github.com/BibliothecaDAO/codename-bobby-realms) that uses base64 encoded PNGs inside `<image>` tags.

### Code

Now that we've got the theory covered, let's get cooking.

We can bootstrap our coding efforts using a template from the wonderful [Cairo Wizard from OpenZeppelin](https://wizard.openzeppelin.com/cairo#erc721). This deals with the boring part of a NFT implementation. We need to modify the template slightly. Instead of embedding the ERC721MixinImpl we'll only use the ERC721Impl and do the IERC721Metadata functions (`name`, `symbol` and most importantly `token_uri`) ourselves. That also means we can delete the ERC721 initializer from the constructor.

The `token_uri` is the part we're interested in. This function must return a string representing the whole NFT combining the techniques described above. Because the string contains double quotes (`"`) and curly braces (`{}`), we have to escape them properly in Cairo. That means using `\"` for a literal double quote and `{{` or `}}` for curlies in the `format!` macro.

As usual, the [code is open source](https://github.com/milancermak/nouns-glasses), so go check it out.

### Free mint

Not only is the whole code freely available, minting the NFT is free as well. Since this is a educational project, there's no frontend, you'll have to use Starkscan to call the `mint` function. On the [contract page](https://starkscan.co/contract/0x03e4b9cd4a127040fd29a0373bd62ca6a3135cc6a9b3a10588ce664ac6478e9c#read-write-contract-sub-write), connect your wallet (1) and then execute the `mint` function (2). There's no limit to the number of times you can mint, but note all images from a single address will look the same as the address is used to generate the rim color.

Happy minting.
