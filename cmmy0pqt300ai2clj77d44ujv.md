---
title: "Protecting Your Users with NSFWJS"
datePublished: 2026-03-19T22:04:32.635Z
cuid: cmmy0pqt300ai2clj77d44ujv
slug: protecting-your-users-with-nsfwjs
cover: https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/b47cddeb-f4f2-49cc-b31d-8d69ed479b3d.png
tags: content-moderation, explicit-content-filtering, indecent-content-filtering

---

In the modern web, user-generated content (UGC) is everywhere. While it drives engagement, it also brings a major challenge: **content moderation**. How do you prevent inappropriate images from being uploaded or displayed without building a massive, expensive backend infrastructure?

Enter [**NSFWJS**](https://www.npmjs.com/package/nsfwjs), a powerful, open-source JavaScript library that brings indecent content checking directly to the client's browser.

[**Try the live demo**](https://nsfwjs.com/)

![Preview of the NSFWJS demo live classification.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/8da5f129-781e-4df0-aea9-0c28133dec88.gif align="center")

## Why Client-Side Moderation?

Traditionally, moderation happens on the server. While powerful services like [AWS Rekognition](https://aws.amazon.com/rekognition/) or [Google Cloud Vision](https://cloud.google.com/vision) are industry standards, they come with trade-offs. You have to upload every user image to their cloud, which introduces latency, privacy concerns, and recurring per-image costs.

**NSFWJS flips the script.** By leveraging [TensorFlow.js](https://js.tensorflow.org/), the moderation happens on the user's device. This offers three massive advantages:

1.  **Privacy**: Unlike cloud APIs, the image never has to leave the user's device. This is a huge win for user trust and data privacy compliance.
    
2.  **Scalability**: You offload the heavy lifting (machine learning inference) to the client's hardware. Your servers stay lean, and you avoid the "bill-per-image" model.
    
3.  **Immediate Feedback**: Users get instant results without waiting for a round-trip to a remote server. This enables unique use cases like **real-time video feed or live webcam analysis**, where you can classify several frames per second to provide continuous moderation.
    

## Categorizing the Web

NSFWJS doesn't just give you a "Yes/No" answer. It provides probabilities across five distinct classes so that you can choose the level of moderation and intensity you want.

*   😐 **Neutral**: Everyday, safe-for-work images.
    
*   🎨 **Drawing**: Safe-for-work drawings and anime.
    
*   🔥 **Sexy**: Sexually explicit images, but not quite pornography.
    
*   🔞 **Hentai**: Hentai and pornographic drawings.
    
*   🔞 **Porn**: Pornographic images and sexual acts.
    

## Getting Started in 30 Seconds

Integrating NSFWJS is incredibly straightforward. Here’s how you can classify an image element in your web app:

```javascript
import * as nsfwjs from "nsfwjs";

// Load the model (MobileNetV2 is the default)
const model = await nsfwjs.load();

// Classify an image element, video, or canvas
const img = document.getElementById("user-upload-preview");
const predictions = await model.classify(img);

console.log("Predictions:", predictions);
```

### Advanced: Optimizing for Production (Tree-Shaking)

If you're building a production app, you might want to keep your bundle as small as possible. By using the `nsfwjs/core` entry point, you can manually register only the models you need:

```javascript
import { load } from "nsfwjs/core";
import { MobileNetV2Model } from "nsfwjs/models/mobilenet_v2";

// Only bundles and loads the specific model you want
const model = await load("MobileNetV2", {
  modelDefinitions: [MobileNetV2Model],
});

const predictions = await model.classify(img);
```

*For a comprehensive, real-world React implementation using Web Workers and local caching in the browser, check out the* [*Demo*](https://nsfwjs.com/) *in the* [*GitHub repository*](https://github.com/infinitered/nsfwjs/tree/master/examples/nsfw_demo)*.*

## Shared Responsibility: Frontend + Backend

While client-side moderation is a powerful first line of defense, it should **never be your only line of defense**. A savvy user can always bypass frontend code. For a robust system, moderation is a shared responsibility:

*   **Frontend**: Provides instant feedback to the user, reduces server load, and acts as a filter for the majority of uploads.
    
*   **Backend**: Acts as the ultimate source of truth. You should always implement a middleware or a server-side hook (using `@tensorflow/tfjs-node` with `NSFWJS`) to verify images before they are permanently stored or served to other users.
    

### Backend Verification (Node.js)

To ensure your moderation is tamper-proof, you should also verify the content on your server. Here is a quick example of how to classify an image in a Node.js environment:

```javascript
const tf = require("@tensorflow/tfjs-node");
const nsfw = require("nsfwjs");

async function verifyImage(imageBuffer) {
  // Load the model (ideally from a local file)
  const model = await nsfw.load(); 

  // Convert buffer to a 3D Tensor
  const image = await tf.node.decodeImage(imageBuffer, 3);

  // Classify and dispose of the tensor to prevent memory leaks
  const predictions = await model.classify(image);
  image.dispose();

  return predictions;
}
```

## Performance & Flexibility

Whether you're building a lightweight mobile site or a heavy-duty web app, NSFWJS has you covered:

*   **Multiple Models**: Choose between `MobileNetV2` (fast & small), `MobileNetV2Mid` (balanced), or `InceptionV3` (most accurate but also the heaviest).
    
*   **Tree-Shaking**: Using the `nsfwjs/core` entry point allows you to bundle only the models you need, keeping your JS payload tiny.
    
*   **Host Your Own Models**: While models can be bundled into your JS, for optimal performance, [host the model files](https://github.com/infinitered/nsfwjs?tab=readme-ov-file#host-your-own-model) directly on your CDN. This allows you to load model binaries directly and enables browsers to cache them separately, reducing your initial JS bundle size.
    
*   **Node.js Support**: Need to do some server-side checks, too? NSFWJS works perfectly with `@tensorflow/tfjs-node`.
    
*   **Backend Selection**: It supports `WebGL`, `WASM`, and even the new `WebGPU` for blazing-fast performance.
    

## Conclusion

As developers, we have a responsibility to create safe digital spaces. NSFWJS makes it easy to add a layer of protection to your applications without sacrificing user privacy or breaking the bank on server costs.

**Note on Accuracy:** Machine learning is never 100% perfect. NSFWJS is highly accurate (~90-93%), but false positives can happen. If you encounter an image that is misclassified, consider [**reporting it on GitHub**](https://github.com/infinitered/nsfwjs/issues) so that it can be used to continue improving the model for everyone.

Give it a star on [**GitHub**](https://github.com/infinitered/nsfwjs) and join the community of contributors making the web a safer place!