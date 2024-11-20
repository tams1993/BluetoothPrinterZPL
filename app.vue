<template>
  <div class="p-4">
    <div class="text-center mb-4">
      <span :class="[isSupported ? 'text-green-500' : 'text-red-500']">
        {{ isSupported ? 'Bluetooth Web API Supported' : 'Bluetooth Web API Not Supported' }}
      </span>
    </div>
    <div v-if="isSupported" class="mb-4 flex justify-center">
      <button @click="requestDevice" class="btn btn-primary">Request Bluetooth Device</button>
    </div>
    <div v-if="device" class="text-center mb-4">
      <p>Device Name: {{ device.name }}</p>
    </div>
   
    <div v-if="isConnected" class="mb-4 flex flex-col items-center">
      <p>Connected</p>
      <input v-model="customText" placeholder="Enter custom text" class="input input-bordered mb-2 w-full max-w-sm" />
      <input type="file" @change="onImageSelected" accept="image/*" class="mb-2" />
      <button @click="printImageAndText" class="btn btn-primary">Print Image & Text</button>
    </div>
  </div>
</template>


<script setup lang="ts">
import { useBluetooth } from '@vueuse/core';
import { Jimp } from 'jimp';
import { Buffer } from 'buffer';
import { Label, Text, Graphic, GraphicData, PrintDensity, PrintDensityName, FontFamily, FontFamilyName, Spacing, Image } from 'jszpl';

// Bluetooth setup
const {
  isConnected,
  isSupported,
  device,
  server,
  requestDevice,
} = useBluetooth({
  acceptAllDevices: true,
  optionalServices: [
    '000018f0-0000-1000-8000-00805f9b34fb',
    '0000180f-0000-1000-8000-00805f9b34fb',
    'generic_access',
    'generic_attribute',
  ],
});

const customText = ref('');
const selectedImage = ref<File | null>(null);
const graphic = new Graphic();

// Helper function to split data into chunks
const chunkArray = (arr: Uint8Array, chunkSize: number): Uint8Array[] => {
  const result: Uint8Array[] = [];
  for (let i = 0; i < arr.length; i += chunkSize) {
    result.push(arr.slice(i, i + chunkSize));
  }
  return result;
};

const onImageSelected = async (event: Event) => {
  const input = event.target as HTMLInputElement;
  if (input.files && input.files.length > 0) {
    selectedImage.value = input.files[0];
    await processImage(selectedImage.value);
    console.log('Graphic data processed successfully');
  }
};

const printImageAndText = async () => {
  try {
    console.log('Starting print process...');
    const label = new Label();
    label.content.push(graphic);

    label.width = 80;
    label.printDensity = new PrintDensity(PrintDensityName['8dpmm']);
    label.padding = new Spacing(10, 20, 0, 20); // top, right, bottom, left margins

    const textField = new Text();
    textField.text = customText.value;
    textField.fontFamily = new FontFamily(FontFamilyName['A']);
    textField.characterWidth = 20;
    textField.characterHeight = 40;

    label.content.push(textField);
    const zplCommand = label.generateZPL();
    console.log('Generated ZPL Command:', zplCommand);

    const encoder = new TextEncoder();
    const data = encoder.encode(zplCommand);

    const service = await server.value.getPrimaryService("000018f0-0000-1000-8000-00805f9b34fb");
    const characteristic = await service.getCharacteristic('00002af1-0000-1000-8000-00805f9b34fb');

    if (characteristic) {
      const CHUNK_SIZE = 512;
      const chunks = chunkArray(data, CHUNK_SIZE);
      for (const chunk of chunks) {
        await characteristic.writeValue(chunk);
        console.log('Sent chunk:', chunk.length, 'bytes');
      }
      alert('Print job sent successfully!');
    } else {
      alert('Failed to get Bluetooth characteristic. Check the device and try again.');
    }
  } catch (error) {
    console.error('Printing error:', error);
    alert(`Printing failed: ${error.message}`);
  }
};

const processImage = async (file: File) => {
  try {
    const base64 = await fileToBase64(file);
    const image = await Jimp.read(base64);
    const { width: originalWidth, height: originalHeight } = image;
    let imageBits: number[] = [];

    // Calculate the new height to maintain aspect ratio
    const desiredWidth = 200;
    const aspectRatio = originalHeight / originalWidth;
    const calculatedHeight = desiredWidth * aspectRatio;

    graphic.width = desiredWidth;
    graphic.height = calculatedHeight;
    graphic.margin = new Spacing(200,20,0,0)

    image.scan(0, 0, originalWidth, originalHeight, function(x, y, idx) {
      const red = this.bitmap.data[idx + 0];
      const green = this.bitmap.data[idx + 1];
      const blue = this.bitmap.data[idx + 2];
      const alpha = this.bitmap.data[idx + 3];

      let value = 0;
      if (alpha !== 0) {
        const gray = (red + green + blue) / 3;
        value = (gray < 180) ? 1 : 0;
      }
      imageBits.push(value);
    });

    const graphicData = new GraphicData(originalWidth, originalHeight, imageBits);
    graphic.data = graphicData;
  } catch (error) {
    console.error('Error processing image:', error);
    throw error;
  }
};

// Helper function to convert File to Base64
const fileToBase64 = (file: File): Promise<string> => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      if (typeof reader.result === 'string') {
        resolve(reader.result);
      } else {
        reject(new Error('Failed to convert file to Base64'));
      }
    };
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
};
</script>