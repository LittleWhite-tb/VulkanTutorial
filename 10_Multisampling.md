## Introduction

Our program can now load multiple levels of detail for textures which fixes artifacts when rendering objects far away from the viewer. The image is now a lot smoother, however on closer inspection you will notice jagged saw-like patterns along the edges of drawn geometric shapes. This is especially visible in one of our early programs when we rendered a quad:

![](/images/texcoord_visualization.png)

This undesired effect is called "aliasing" and it's a result of a limited numbers of pixels that are available for rendering. Since there are no displays out there with unlimited resolution, it will be always visible to some extent. There's a number of ways to fix this and in this chapter we'll focus on one of the more popular ones: [Multisample anti-aliasing](https://en.wikipedia.org/wiki/Multisample_anti-aliasing) (MSAA).

In ordinary rendering, the pixel color is determined based on a single sample point which in most cases is the center of target pixel on screen. If part of the drawn line passes through a certain pixel but doesn't cover the sample point, that pixel will be left blank, leading to the jagged "staircase" effect.

![](/images/aliasing.png)

What MSAA does is it uses multiple sample points per pixel (hence the name) to determine its final color. As one might expect, more samples lead to better results, however it is also more computationally expensive.

![](/images/antialiasing.png)

In our implementation, we will focus on using the maximum available sample count. Depending on your application this may not always be the best approach and it might be better to use less samples for the sake of higher performance if the final result meets your quality demands.


## Getting available sample count

Let's start off by determining how many samples our hardware can use. Most modern GPUs support at least 8 samples but this number is not guaranteed to be the same everywhere. We'll keep track of it by adding a new class member:

```c++
...
VkSampleCountFlagBits msaaSamples = VK_SAMPLE_COUNT_1_BIT;
...
```

By default we'll be using only one sample per pixel which is equivalent to no multisampling, in which case the final image will remain unchanged. The exact maximum number of samples can be extracted from `VkPhysicalDeviceProperties` associated with our selected physical device. We're using a depth buffer, so we have to take into account the sample count for both color and depth - the lower number will be the maximum we can support. Add a function that will fetch this information for us:

```c++
VkSampleCountFlagBits getMaxUsableSampleCount() {
    VkPhysicalDeviceProperties physicalDeviceProperties;
    vkGetPhysicalDeviceProperties(physicalDevice, &physicalDeviceProperties);

    VkSampleCountFlags counts = std::min(physicalDeviceProperties.limits.framebufferColorSampleCounts, physicalDeviceProperties.limits.framebufferDepthSampleCounts);
    if (counts & VK_SAMPLE_COUNT_64_BIT) { return VK_SAMPLE_COUNT_64_BIT; }
    if (counts & VK_SAMPLE_COUNT_32_BIT) { return VK_SAMPLE_COUNT_32_BIT; }
    if (counts & VK_SAMPLE_COUNT_16_BIT) { return VK_SAMPLE_COUNT_16_BIT; }
    if (counts & VK_SAMPLE_COUNT_8_BIT) { return VK_SAMPLE_COUNT_8_BIT; }
    if (counts & VK_SAMPLE_COUNT_4_BIT) { return VK_SAMPLE_COUNT_4_BIT; }
    if (counts & VK_SAMPLE_COUNT_2_BIT) { return VK_SAMPLE_COUNT_2_BIT; }

    return VK_SAMPLE_COUNT_1_BIT;
}
```

We will now use this function to set the `msaaSamples` variable during the physical device selection process. For this, we have to slightly modify the `pickPhysicalDevice` function:

```c++
void pickPhysicalDevice() {
            ...
            if (isDeviceSuitable(device)) {
                physicalDevice = device;
                msaaSamples = getMaxUsableSampleCount();
                break;
            }
            ...
}
```

Next, update `createImage` function to allow us to specify the number of used samples by adding a `numSamples` parameter - this will become important later:

```c++
void createImage(uint32_t width, uint32_t height, uint32_t mipLevels, VkSampleCountFlagBits numSamples, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory) {
    ...
    imageInfo.samples = numSamples;
    ...
```

For now, update all calls to this function using `VK_SAMPLE_COUNT_1_BIT` - we will be replacing this with proper values as we progress with implementation:

```c++
createImage(swapChainExtent.width, swapChainExtent.height, 1, VK_SAMPLE_COUNT_1_BIT, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImages[i], depthImagesMemory[i]);
...
createImage(texWidth, texHeight, mipLevels, VK_SAMPLE_COUNT_1_BIT, VK_FORMAT_R8G8B8A8_UNORM, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
```

## Setting up render targets

In MSAA, each pixel is sampled in an offscreen buffer which is then rendered to the screen. These new buffers are slightly different from regular images we've been rendering to - they have to be able to store more than one sample per pixel. Once a multisampled buffer is created, it has to be attached to the default framebuffer (which stores only a single sample per pixel). This is why we have to create additional render targets and modify our current drawing process. Add following class members:

```c++
...
std::vector<VkImage> colorImages;
std::vector<VkDeviceMemory> colorImagesMemory;
std::vector<VkImageView> colorImagesView;

std::vector<VkImage> depthMsaaImages;
std::vector<VkDeviceMemory> depthMsaaImagesMemory;
std::vector<VkImageView> depthMsaaImagesView;
...
```

We will now create a multisampled color buffer. Add a `createColorResources` function and note that we're using `msaaSamples` here as a function parameter to `createImage`. We're also using only one mip level, since this is enforced by the Vulkan specification in case of images with more than one sample per pixel:

```c++
void createColorResources() {
    VkFormat colorFormat = swapChainImageFormat;

    colorImages.resize(swapChainImages.size());
    colorImagesMemory.resize(swapChainImages.size());
    colorImageViews.resize(swapChainImages.size());

    for (size_t i = 0; i < swapChainImageViews.size(); i++) {
        createImage(swapChainExtent.width, swapChainExtent.height, 1, msaaSamples, colorFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT | VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, colorImages[i], colorImagesMemory[i]);
        colorImageViews[i] = createImageView(colorImages[i], colorFormat, VK_IMAGE_ASPECT_COLOR_BIT, 1);

        transitionImageLayout(colorImages[i], colorFormat, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL, 1);
    }
}
```

For consistency, call the function right before `createDepthResources`:

```c++
void initVulkan() {
    ...
    createColorResources();
    createDepthResources();
    ...
}
```

You may notice that the newly created color image uses a transition path from `VK_IMAGE_LAYOUT_UNDEFINED` to `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` which is a new case for us to handle. Let's update `transitionImageLayout` function to take this into account:

```c++
void transitionImageLayout(VkImage image, VkFormat format, VkImageLayout oldLayout, VkImageLayout newLayout, uint32_t mipLevels) {
    ...
    else if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL) {
        barrier.srcAccessMask = 0;
        barrier.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_READ_BIT | VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
        sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
        destinationStage = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    }
    else {
        throw std::invalid_argument("unsupported layout transition!");
    }
    ...
}
```

Now that we have multisampled color buffer in place it's time to take care of depth. Modify `createDepthResources` and add creation steps for a multisampled depth buffer:

```c++
void createDepthResources() {
    ...
    depthMsaaImages.resize(swapChainImages.size());
    depthMsaaImagesMemory.resize(swapChainImages.size());
    depthMsaaImagesView.resize(swapChainImages.size());
    ...

    for (size_t i = 0; i < swapChainImageViews.size(); i++) {
        ...
        createImage(swapChainExtent.width, swapChainExtent.height, 1, msaaSamples, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthMsaaImages[i], depthMsaaImagesMemory[i]);
        depthMsaaImagesView[i] = createImageView(depthMsaaImages[i], depthFormat, VK_IMAGE_ASPECT_DEPTH_BIT, 1);

        transitionImageLayout(depthMsaaImages[i], depthFormat, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL, 1);
    }
}
```

We have now created a couple of new Vulkan resources, so let's not forget to release them when necessary:

```c++
void cleanupSwapChain() {
    for (size_t i = 0; i < swapChainImages.size(); i++) {
        vkDestroyImageView(device, colorImageViews[i], nullptr);
        vkDestroyImage(device, colorImages[i], nullptr);
        vkFreeMemory(device, colorImagesMemory[i], nullptr);
        vkDestroyImageView(device, depthMsaaImagesView[i], nullptr);
        vkDestroyImage(device, depthMsaaImages[i], nullptr);
        vkFreeMemory(device, depthMsaaImagesMemory[i], nullptr);
        ...
    }
    ...
}
```

With only a few simple steps we created additional buffers and image views necessary for multsampling - it's now time to put it all together and see the results!

## Adding new attachments

Let's take care of the render pass first. Modify `createRenderPass` and update color and depth attachment creation info structs:

```c++
void createRenderPass() {
    ...
    colorAttachment.samples = msaaSamples;
    colorAttachment.finalLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    ...
    depthAttachment.samples = msaaSamples;
    ...
```

Apart from the obvious change that tells both color and depth to use more samples, you'll notice an update to the `finalLayout` parameter for the color attachment. This is because a multisampled image is a fairly complex structure containing more information than a regular image. One of the consequences of this is that it cannot be processed by a sampler, so it cannot be drawn directly to the screen. For that reason, a multisampled image has to be translated (resolved) to a regular image before it can be used. This also applies to the depth buffer, so we need to define two additional attachments:

```c++
    ...
    VkAttachmentDescription colorAttachmentResolve = {};
    colorAttachmentResolve.format = swapChainImageFormat;
    colorAttachmentResolve.samples = VK_SAMPLE_COUNT_1_BIT;
    colorAttachmentResolve.loadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    colorAttachmentResolve.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    colorAttachmentResolve.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    colorAttachmentResolve.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    colorAttachmentResolve.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    colorAttachmentResolve.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;

    VkAttachmentDescription depthAttachmentResolve = {};
    depthAttachmentResolve.format = findDepthFormat();
    depthAttachmentResolve.samples = VK_SAMPLE_COUNT_1_BIT;
    depthAttachmentResolve.loadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    depthAttachmentResolve.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    depthAttachmentResolve.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    depthAttachmentResolve.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    depthAttachmentResolve.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    depthAttachmentResolve.finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
    ...
```

The render pass now has to be instructed to perform color image resolution. Create a new attachment reference that will point to the color buffer which will serve as the resolve target:

```c++
    ...
    VkAttachmentReference colorAttachmentResolveRef = {};
    colorAttachmentResolveRef.attachment = 2;
    colorAttachmentResolveRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    ...
```

Set the `pResolveAttachments` subpass struct member to point to the newly created attachment reference. This is enough to let the render pass define a multisample resolve operation which will let us render the image to screen:

```
    ...
    subpass.pResolveAttachments = &colorAttachmentResolveRef;
    ...
```

Now update render pass info struct with new attachments:

```c++
    ...
    std::array<VkAttachmentDescription, 4> attachments = {colorAttachment, depthAttachment, colorAttachmentResolve, depthAttachmentResolve };
    ...
```

With render pass in place, modify `createFrameBuffers` and add new image views to the list:

```c++
void createFrameBuffers() {
        ...
        std::array<VkImageView, 4> attachments = {
            colorImageViews[i],
            depthMsaaImagesView[i],
            swapChainImageViews[i],
            depthImagesView[i]
        };
        ...
}
```

Finally, tell the newly created pipeline to use more than one sample by modifying `createGraphicsPipeline`:

```c++
void createGraphicsPipeline() {
    ...
    multisampling.rasterizationSamples = msaaSamples;
    ...
}
```

Now run your program and you should see the following:

![](/images/multisampling.png)

Just like with mipmapping, the difference may not be apparent straight away. On a closer look you'll notice that the edges on the roof are not as jagged anymore and the whole image seems a bit smoother compared to the original.

![](/images/multisampling_comparison.png)

The difference is more noticable when looking up close at one of the edges:

![](/images/multisampling_comparison2.png)


## Conclusion

It has taken a lot of work to get to this point, but now you finally have a good
base for a Vulkan program. The knowledge of the basic principles of Vulkan that
you now possess should be sufficient to start exploring more of the features,
like:

* Push constants
* Instanced rendering
* Dynamic uniforms
* Separate images and sampler descriptors
* Pipeline cache
* Multi-threaded command buffer generation
* Multiple subpasses
* Compute shaders

The current program can be extended in many ways, like adding Blinn-Phong
lighting, post-processing effects and shadow mapping. You should be able to
learn how these effects work from tutorials for other APIs, because despite
Vulkan's explicitness, many concepts still work the same.

[C++ code](/code/29_multisampling.cpp) /
[Vertex shader](/code/26_shader_depth.vert) /
[Fragment shader](/code/26_shader_depth.frag)