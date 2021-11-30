# This repo demonstrate a minimal Teams video app.

## Install the video app in Teams
1. Host the app directory in a public accessible HTTPS server. You can use github page as the host.
2. Replace the `name`, `appId`, and `contentUrl` in `meta/manifest.json`.
    1. The contentUrl should point to your app directory, like `https://lubobill1990.github.io/Teams-VideoApp-example/app/`
    2. appId can be any unique GUID
3. zip the meta directory, choose the zip file after clicking Upload a custom app
4. Go to a teams meeting, enable the video, and activate the video app.


## Develop video app in browser

1. open terminal
2. `cd` to the directory of README.md
3. run `yarn install`
4. run `yarn start-app`
5. run `yarn start-container`
6. open `https://localhost:9000/index.html` in your browser
7. change `videoFrameHandler` function in `app/index.js`

## WASM

1. using emsdk 2.0.30, reference `https://emscripten.org/docs/getting_started/downloads.html` for installing and change version to 2.0.30 by `emsdk activate 2.0.30`
2. compile to wasm by `emcc -o wasm-ccall.js -s TOTAL_MEMORY=64MB -s "EXPORTED_RUNTIME_METHODS=['ccall', 'cwrap']" -O3 wasm-ccall.cpp`

## About color conversion and WebGL rendering in this sample app
This app contains sample code for color conversion and WebGL rendering.
### Some overall results and understandings
1. With current processing pipeline, a full round (NV12 -> process -> NV12) is ~28ms.
2. The app processing pipeline is critical to performance. Need to carefully choose the intermediate datatype, reduce data redundant as much as possible. 
3. Besides convert color space, the render/GPU<->CPU copy are more time consuming, we should focus on how to leverage more GPU to reduce data transfer cost and CPU usage. E.g. convert back to nv12 in shader, then copy back the nv12 array only. 
4. It would highly increase the throughput if there are some mechanisms to enable direct access texture from CPU. I have not found such mechanism in WebGL for now.

### Test environment
Resolution: 1280x720 
Laptop CPU: i7-106100U @ 1.80GHz, GPU: Intel UHD Graphics (something like 630). 

### This sample performs following operations
1. Take NV12 input(javascript TypedArray) convert to RGBAf (javascript TypedArray). 
2. Copy RGBAf to WebGL texture as input. 
3. WebGL render: The effect is simple, just flip X and Y.
4. Copy RGBAU8 (javascript TypedArray) from WebGL texture as output
5. Convert RGBAU8 back to NV12 and overwrite the input NV12. 
6. For RGBAf & RGBAU8(marked as green), they are accessible from javascript (as TypedArray) and wasm (as native pointer). The RGBAf could be more suitable then U8 as inference input. 

### Known issues
1. Only 1280*720 is supported yet, the resolution is hard-coded for now. 
2. Dynamic resolution changing is not supported yet. 
3. For RGBA -> NV12, I think there are something wrong with the prebuilt wasm, probably caused by emscripten compiler, which cause it is slower than NV12->RGBAf. 
4. Code branch for async copyback have some bugs, which might cause GPU stall on certain execution sequence. 
