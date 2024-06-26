### Description of the Code

This code is designed to visualize the intermediate layers of a Convolutional Neural Network (CNN) using Grad-CAM (Gradient-weighted Class Activation Mapping). Grad-CAM is a technique that helps understand which parts of an image influence the decision of a CNN model by highlighting the important regions in the input image. Here's a detailed breakdown of the code:

#### Gard-CAM Architecture

![GradCAM-architecture-1536x733](https://github.com/HR-Fahim/CNNs-Grad-CAM-on-CIFAR-10/assets/66734379/d6c987de-f83c-4120-b1b4-0436a0be1a10)

#### 1. **Loading and Preprocessing Data**

- **Loading CIFAR-10 Data**: The CIFAR-10 dataset is loaded, which contains 60,000 32x32 color images in 10 different classes.
    ```python
    (train_images, train_labels), (test_images, test_labels) = cifar10.load_data()
    ```

- **Selecting and Preprocessing an Image**: A single image is selected from the training set, resized to 224x224 (to match the VGG16 input size), expanded to fit the model input shape, and preprocessed using the `preprocess_input` function from `tensorflow.keras.applications.vgg16`.
    ```python
    img = train_images[0]
    img = cv2.resize(img, (224, 224))
    img = np.expand_dims(img, axis=0)
    img = preprocess_input(img)
    ```

#### 2. **Loading and Modifying the VGG16 Model**

- **Loading VGG16**: The VGG16 model is loaded with pre-trained weights from ImageNet, excluding the top classification layer.
    ```python
    base_model = VGG16(weights='imagenet', include_top=False)
    ```

- **Adding Custom Layers**: A global average pooling layer, a dense layer with 1024 units and ReLU activation, and a final dense layer with 10 units (for CIFAR-10 classification) and softmax activation are added.
    ```python
    x = base_model.output
    x = tf.keras.layers.GlobalAveragePooling2D()(x)
    x = tf.keras.layers.Dense(1024, activation='relu')(x)
    predictions = tf.keras.layers.Dense(10, activation='softmax')(x)
    model = tf.keras.models.Model(inputs=base_model.input, outputs=predictions)
    ```

#### 3. **Defining the Grad-CAM Function**

- **Creating a Gradient Model**: The model is wrapped to create a new model (`grad_model`) that outputs the specified convolutional layer’s output and the final predictions.
    ```python
    grad_model = tf.keras.models.Model(model.input, [model.get_layer(layer_name).output, model.output])
    ```

- **Gradient Calculation**: Gradients of the output with respect to the chosen convolutional layer are calculated.
    ```python
    with tf.GradientTape() as tape:
        conv_outputs, predictions = grad_model(img)
        loss = predictions[:, np.argmax(predictions[0])]
    
    output = conv_outputs[0]
    grads = tape.gradient(loss, conv_outputs)[0]
    ```

- **Guided Gradients**: Positive gradients are extracted by gating with ReLU.
    ```python
    gate_f = tf.cast(output > 0, 'float32')
    gate_r = tf.cast(grads > 0, 'float32')
    guided_grads = gate_f * gate_r * grads
    ```

- **Weight Calculation**: The gradients are averaged across the spatial dimensions to obtain weights.
    ```python
    weights = tf.reduce_mean(guided_grads, axis=(0, 1))
    ```

- **Generating CAM**: A class activation map is generated by a weighted combination of the convolutional layer outputs.
    ```python
    cam = np.ones(output.shape[0: 2], dtype = np.float32)
    for i, w in enumerate(weights):
        cam += w * output[:, :, i]
    cam = cv2.resize(cam.numpy(), (224, 224))
    cam = np.maximum(cam, 0)
    heatmap = (cam - cam.min()) / (cam.max() - cam.min())
    ```

- **Overlaying CAM on the Image**: The CAM is overlaid on the original image using a colormap.
    ```python
    image = img[0, :]
    image -= image.min()
    image = np.minimum(image, 255)
    
    cam = cv2.applyColorMap(np.uint8(255*heatmap), cv2.COLORMAP_JET)
    cam = np.float32(cam) + np.float32(image)
    cam = 255 * cam / np.max(cam)
    ```

#### 4. **Creating a GIF with Layer Visualizations**

- **Layer List**: A list of layers to visualize.
    ```python
    layers = [
        'block1_conv1', 'block1_conv2', 
        'block2_conv1', 'block2_conv2', 
        'block3_conv1', 'block3_conv2', 'block3_conv3',
        'block4_conv1', 'block4_conv2', 'block4_conv3',
        'block5_conv1', 'block5_conv2', 'block5_conv3'
    ]
    ```

- **Generating and Saving GIF Frames**: For each layer, Grad-CAM is applied, the resulting image is combined with a black canvas, and text detailing the layer and output size is added.
    ```python
    frames = []
    for layer in layers:
        cam = get_grad_cam(model, img, layer)
        cam_img = Image.fromarray(cam)
        
        canvas_width = cam_img.width + 40
        canvas_height = cam_img.height + 100
        new_img = Image.new('RGB', (canvas_width, canvas_height), (0, 0, 0))
        offset = ((canvas_width - cam_img.width) // 2, 20)
        new_img.paste(cam_img, offset)
        
        draw = ImageDraw.Draw(new_img)
        font = ImageFont.load_default()
        layer_details = f"Layer: {layer}, Output size: {model.get_layer(layer).output.shape[1:]}"
        text_width, text_height = draw.textsize(layer_details, font=font)
        text_position = ((canvas_width - text_width) // 2, cam_img.height + 30)
        draw.text(text_position, layer_details, font=font, fill=(255, 255, 255))
        
        frames.append(new_img)
    frames[0].save('grad_cam.gif', save_all=True, append_images=frames[1:], duration=500, loop=0)
    ```

- **Displaying the GIF**: The resulting GIF is displayed in the notebook.
    ```python
    from IPython.display import Image as IPImage
    IPImage(filename='grad_cam.gif')
    ```
- **Output Visualization**: Loaded output as GIF.

     ![grad_cam](https://github.com/HR-Fahim/CNNs-Grad-CAM-on-CIFAR-10/assets/66734379/b724002f-ddc1-44e9-89e6-fc1b817d87c4)

### Explanation of Grad-CAM

Grad-CAM (Gradient-weighted Class Activation Mapping) is a technique to visualize and understand the decisions made by Convolutional Neural Networks (CNNs). Here's how it works:

1. **Identify the Target Class**: Choose the class for which  want to visualize the activations.
2. **Compute Gradients**: Calculate the gradient of the class score with respect to the feature maps of a specific convolutional layer. This indicates how important each spatial location in the feature maps is for the class.
3. **Weight the Feature Maps**: Average the gradients spatially to obtain a weight for each feature map channel.
4. **Generate the Activation Map**: Compute a weighted combination of the forward activation maps using these weights.
5. **Apply ReLU**: Apply a ReLU to the resulting map to focus on the positive influences.
6. **Overlay on Original Image**: Overlay this class activation map on the original image to highlight the regions important for the classification decision.

This technique is particularly useful for:
- **Model Interpretation**: Understanding what parts of an image are important for a model's prediction.
- **Debugging**: Identifying potential issues in model learning, such as overfitting to specific features.
- **Explaining Decisions**: Providing visual explanations for model predictions, which can be crucial for applications requiring transparency and trust, such as in medical imaging.

By using Grad-CAM, one can gain insights into the inner workings of CNNs and ensure that they are making decisions based on relevant features of the input data.
