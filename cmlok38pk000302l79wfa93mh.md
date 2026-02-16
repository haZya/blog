---
title: "Elevate Your UI with Dynamic Text Shadows in React with ShineJS"
datePublished: Mon Feb 16 2026 02:29:30 GMT+0000 (Coordinated Universal Time)
cuid: cmlok38pk000302l79wfa93mh
slug: elevate-your-ui-with-dynamic-text-shadows-in-react-with-shinejs
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1771207393967/752f45cb-1dca-4ff3-88f5-ead2df62058f.png
tags: reactjs, neumorphism, shinejs, text-shadow

---

As developers, we're always looking for that extra "pop" to make our interfaces stand out. Whether you're a fan of **neumorphism** or just want to add a bit of tactile depth to your typography, static shadows often fall flat. They don't react to the environment, and they certainly don't feel "alive."

That’s why I built [**ShineJS**](https://github.com/haZya/shinejs), a modern, lightweight library designed to bring dynamic, light-reactive shadows to your React and Next.js projects.

In this article, I’ll show you how to get started with the core component and provide an interactive playground where you can experiment with **neumorphic-text** effects in real-time.

## What is ShineJS?

[ShineJS](https://github.com/haZya/shinejs) is an ESM-only TypeScript library based on the initial work from [bigspaceship/shine.js](https://github.com/bigspaceship/shine.js) that calculates and injects multi-layered shadows based on a virtual light source. By tracking the mouse position (or a fixed point), it creates a sense of physical presence for your text and UI elements.

It's particularly effective for:

* **Neumorphism** aesthetics where soft, directional shadows define the UI.
    
* Hero sections that need a premium, interactive feel.
    
* Visualizing depth in typography-heavy designs.
    

## Quick Start: The `<Shine />` Component

The easiest way to add a shine effect is to use the high-level React component. It handles the ref management and updates automatically.

### Installation

```bash
npm install @hazya/shinejs
```

### Basic Usage

Here is a simple example of a heading that follows your mouse:

```javascript
import { Shine } from "@hazya/shinejs/react";

export function HeroHeading() {
  return (
    <div className="bg-slate-100 p-20 flex justify-center">
      <Shine
        as="h1"
        className="text-6xl font-black text-slate-200 uppercase tracking-tighter"
        options={{
          light: {
            position: "followMouse",
            intensity: 1.2,
          },
          config: {
            blur: 30,
            opacity: 0.2,
            offset: 0.15,
            shadowRGB: { r: 15, g: 23, b: 42 }, // Slate-900
          },
        }}
      >
        Dynamic Depth
      </Shine>
    </div>
  );
}
```

<iframe src="https://stackblitz.com/github/haZya/shinejs-examples/tree/main/shinejs-simple-demo-react?embed=1&amp;file=src%2FApp.tsx&amp;hideExplorer=1&amp;hideNavigation=1&amp;view=preview" style="width:100%;height:460px;border:0"></iframe>

%[https://stackblitz.com/edit/hazya-shinejs-examples-e2v9gjl3?embed=1] 

## Interactive Playground

I believe the best way to understand a tool is to break it. Below is a live playground that showcases the ShineJS options.

You can tweak the **intensity**, **blur**, and **offset,** etc., to see how the shadow layers interact. This playground uses the `useShine` hook under the hood to give you full control over the rendering logic.

<iframe src="https://stackblitz.com/github/haZya/shinejs-examples/tree/main/shinejs-playground-demo-react?embed=1&file=src%2FApp.tsx&hideExplorer=1&hideNavigation=1&view=preview" style="width:100%;height:980px;border:0"></iframe>

```javascript
"use client";

import { Shine } from "@hazya/shinejs/react";

export function PlaygroundStarter() {
  return (
    <Shine
      as="h1"
      options={{
        light: {
          position: "followMouse",
          intensity: 1,
        },
        config: {
          numSteps: 5,
          opacity: 0.15,
          opacityPow: 1.2,
          offset: 0.15,
          offsetPow: 1.8,
          blur: 40,
          blurPow: 1,
          shadowRGB: { r: 0, g: 0, b: 0 },
        },
      }}
    >
      Shine Playground
    </Shine>
  );
}
```

### Things to Try:

* **Shadow Color:** Change the `shadowRGB` to match your background for a true neumorphic look.
    
* **Opacity Power:** Crank up the `opacityPow` to see how it affects the falloff of the shadow layers.
    
* **Light Position:** Switch from `followMouse` to a `fixed` point to simulate a static light source in your UI.
    

## Typography and Localization

One of the strengths of ShineJS is its adaptability. Because it works with standard CSS text-shadows and box-shadows, it respects your typography choices, including font-family and font-weight.

In this next example, you can see how the effect maintains its integrity across different font families and languages. Whether it's bold Sans-Serif or elegant Serif, the shadows wrap perfectly around every glyph.

<iframe src="https://stackblitz.com/github/haZya/shinejs-examples/tree/main/shinejs-typography-demo-react?embed=1&file=src%2FApp.tsx&hideExplorer=1&hideNavigation=1&view=preview" style="width:100%;height:780px;border:0"></iframe>

```javascript
"use client";

import { Shine } from "@hazya/shinejs/react";

export function TypographyExample() {
  return (
    <Shine
      as="h1"
      style={{
        fontFamily: "Georgia, 'Times New Roman', serif",
        fontWeight: 700,
        fontStyle: "italic",
      }}
      options={{
        light: { position: "followMouse", intensity: 1.15 },
        config: {
          blur: 38,
          offset: 0.12,
          opacity: 0.32,
          shadowRGB: { r: 17, g: 24, b: 39 },
        },
      }}
    >
      Typography Shine
    </Shine>
  );
}
```

### Why this matters for Neumorphism

Neumorphic design relies heavily on the relationship between the light source and the "material" of the UI. By adjusting the typography options alongside ShineJS parameters, you can ensure that your **neumorphic text** remains readable and visually consistent across all device types and screen resolutions.

## Built with Accessibility in Mind

Dynamic text effects can sometimes be a nightmare for screen readers if not handled correctly. When ShineJS splits your text into individual characters to apply granular shadows, it automatically manages the ARIA attributes for you.

The original element receives an `aria-label` containing the full text, while the internal split-text structure is marked with `aria-hidden="true"`. This ensures that screen readers see a single, clean string of text rather than a fragmented series of letters, keeping your **neumorphic text** accessible to everyone.

## Wrapping Up

ShineJS is designed to be unobtrusive yet impactful. It doesn't require heavy canvas rendering or complex WebGL setups. It uses smart CSS injection that follows the physics of light.

If you're building a landing page or a creative portfolio, give `@hazya/shinejs` a try.

* **Documentation:** [shinejs.vercel.app/docs](http://shinejs.vercel.app/docs)
    
* **GitHub:** [haZya/shinejs](https://github.com/haZya/shinejs)
    

Happy coding!