---
title: "Artificial Intelligence in the Browser: Create your own Emoji Translator with Gemma and ONNX"
seoTitle: "Emoji Translator with Gemma: Build with ONNX"
seoDescription: "Create a browser emoji translator using AI models. Learn to fine-tune and optimize with ONNX for client-side deployment"
datePublished: 2025-11-26T14:01:36.712Z
cuid: cmig2pfig000b02jrcokj9wiu
slug: artificial-intelligence-in-the-browser-create-your-own-emoji-translator-with-gemma-and-onnx
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1764162307161/742c2be5-f6a3-47d1-aba1-0a266ac81bb5.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1764165635527/d58f8c9c-ed3d-4711-a07f-38c9a8c11db1.png
tags: tutorials, finetuning, onnx, gemma, ondeviceai, transformersjs

---

Hello Neuralstack community!

Today I'm very excited to present a project that brings the power of large language models (LLMs) directly to your web browser, giving you the opportunity to create something fun and surprisingly useful: an AI-powered emoji translator! We'll be looking at how to fine-tune Google's Gemma 3 270M IT model and then optimize it for client-side inference using the ONNX format.

This project is a fantastic example of how modern AI can be made accessible and efficient by going beyond server-side GPUs and running directly on everyday devices.

## The Challenge: LLMs in Your Browser?

Traditionally, running high-performance LLMs requires significant computing resources, often large GPUs in the cloud. This makes it difficult to directly integrate AI into frontend web applications. However, thanks to advances in model quantization, optimized runtime environments like ONNX, and libraries like Transformers.js, it is becoming increasingly possible to run smaller, yet powerful, LLMs directly in the browser.

The goal of this project was to teach a pre-trained LLM a very special, entertaining task: to convert sentences in natural language into expressive emojis. You can think of it as a personal AI emoji assistant!

### Step 1: Choosing the Brain - Google's Gemma

For this project, we chose Google's [Gemma 3 270M Instruction-Tuned](https://huggingface.co/google/gemma-3-270m-it) model. Why Gemma 270M-IT?

1. **Small and Mighty:** At just 270 million parameters, it's one of the smallest yet most capable LLMs released by Google, making it suitable for resource-constrained environments.
    
2. **Instruction-Tuned:** The "IT" means it's already good at following instructions, which is a great starting point for fine-tuning.
    
3. **Open Access:** Part of Google's commitment to open and responsible AI development.
    

### Step 2: Teaching the AI to Speak Emoji

The core idea is simple: the model should output emojis when given a descriptive sentence. This involves a process called **fine-tuning**. We provide the model with examples like:

* "I am so happy today!" → "😊🎉"
    
* "Let's go to the beach." → "🏖️☀️🌊"
    
* "I love my pet cat." → "😻🐾"
    

By training on enough of these examples, the Gemma model learns the subtle art of emoji selection, understanding context and sentiment.

### Step 3: Optimizing for the Browser with ONNX

Once the Gemma model was fine-tuned to be an emoji expert, the next crucial step was to prepare it for deployment in a web browser. This is where **ONNX (Open Neural Network Exchange)** comes into play.

ONNX is an open standard that defines a common set of operators and a common file format for representing deep learning models. It allows developers to train models in one framework (like PyTorch or TensorFlow), convert them to ONNX format, and then run them with high performance in another environment using an ONNX Runtime.

For web deployment, ONNX models can be executed using libraries like [Transformers.js](https://www.google.com/search?q=https://huggingface.co/docs/transformers.js), which leverages WebAssembly (WASM) and WebGPU (if available) to run the neural network computations directly in your browser's JavaScript environment.

By converting the fine-tuned Gemma model to ONNX, it achieved:

* **Reduced Size:** Often, ONNX conversion involves some level of quantization, further reducing the model's footprint.
    
* **Faster Inference:** ONNX Runtime is highly optimized for various hardware, including the CPUs found in most client devices.
    
* **Browser Compatibility:** It enabled us to bypass server requirements entirely for inference.
    

### Meet the Emoji Translator!

I'm happy to share the result of this project in two Hugging Face repositories and in the NeuralStack | MS Blog Repository under projects:

1. [manuelaschrittwieser/myemoji\_generator-gemma-3-270m-it](https://huggingface.co/manuelaschrittwieser/myemoji_generator-gemma-3-270m-it): This is the model based on **Google's Gemma 3 270M-IT**, a lightweight and efficient language model known for strong instruction-following capabilities in a compact size.
    
2. [manuelaschrittwieser/myemoji-gemma-3-270m-it-onnx](https://huggingface.co/manuelaschrittwieser/myemoji-gemma-3-270m-it-onnx): This is the **fine-tuned and ONNX-optimized emoji translator**! You can find the model files here and explore how to use it with Transformers.js.
    
3. ###### [neuralstack\_blog/projects/my-emoji-generator](https://github.com/MANU-de/neuralstack_blog/tree/master/projects/my-emoji-generator): This is a **small web app** that lets you generate custom emoji styles right in the browser. It uses client-side JavaScript and a web worker to handle generation without blocking the UI.
    

### Try It Out! (Code Example)

The real magic unfolds when you see it in action. Here's a code example that shows how incredibly easy it is to use this model directly in your web application:

```javascript
import { pipeline } from '@xenova/transformers';

async function generateEmojis(text) {
  // Load our ONNX-optimized emoji generator
  const generator = await pipeline(
    'text-generation',
    'manuelaschrittwieser/myemoji-gemma-3-270m-it-onnx'
  );

  // Generate emojis for the given text
  const output = await generator(text, {
    max_new_tokens: 20, // Limit the number of generated tokens (emojis)
    temperature: 0.7, // Adjust for creativity
    do_sample: true,
  });

  return output[0].generated_text;
}

// Example Usage:
(async () => {
  console.log("Input: I am so excited for the party tonight!");
  console.log("Output:", await generateEmojis("I am so excited for the party tonight!"));
  // Expected: 😊🎉🥳

  console.log("\nInput: This weather is just dreadful and rainy.");
  console.log("Output:", await generateEmojis("This weather is just dreadful and rainy."));
  // Expected: ☔🌧️😠
})();
```

### The Future of On-Device AI

This emoji translator project is more than just a novelty; it represents a significant shift in how we can deploy and interact with AI. Imagine:

* **Offline AI:** AI features that work without an internet connection.
    
* **Enhanced Privacy:** User data stays on the device, never sent to a server.
    
* **Lower Latency:** Instant responses because there's no network round trip.
    
* **Reduced Server Costs:** Less reliance on expensive cloud GPUs for inference.
    

We're just scratching the surface of what's possible with smaller, efficient LLMs and browser-based inference. This opens up exciting possibilities for interactive web experiences, creative tools, and accessible AI for everyone.

### What's Next?

I encourage you to explore the [manuelaschrittwieser/myemoji-gemma-3-270m-it-onnx](https://huggingface.co/manuelaschrittwieser/myemoji-gemma-3-270m-it-onnx) repository, fork it, and play around! Think about other small, specific tasks you could teach an LLM to do right in your browser.

---

**— Manuela Schrittwieser, Full-Stack AI Dev 🧑‍💻 & Tech Writer**