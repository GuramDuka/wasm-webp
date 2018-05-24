# wasm-webp
javascript webp decoder

On the fly WebP decoding using WASM and a Service Worker.
Decoding "webp" images in both Chrome and Firefox.
Mini JS library called webp-decoder.js that should work from the main context and from a worker.
It exposes an async function called fetchWebPDecoder returning a WebPDecoder object with a few methods.
Emscripten will create a global Module object, or use yours if you supply it.
As we want to know when everything has been loaded, we need to supply our own.
Though something like var Module = {} works fine directly in the main context or directly in a service worker, it won't work in libraries imported from these places, so the below fixes that — and is very similar to what Emscripten does itself.
What unfortunately doesn't work fine, is calling importScripts from within a JavaScript file loaded via importScripts so that will have to be done manually by the user. ES module should solve all of this, so I am really looking forward to that working in workers!
To get access to the C functions we exposed, we use Emscripten's cwrap.
But as these are quite low level, I wrap it all up in a WebPDecoder class with a few methods.
The constructor takes an ArrayBuffer, for which we create a typed array, an Uint8Array as libwebp excepts an const uint8_t* pointer. We need to copy the data to the WebAssembly heap, so first we allocate a buffer and then we copy the data. For allocation, we allocate byteLength bytes using _malloc and then we copy the data using Module.HEAPU8.set() — all convenience APIs exposed by Emscripten.
Exposing version() is easy as it just returns a number, so we just call the function we got back from cwrap. The info() method is a bit more complicated. In C, we return an integer array (pointer) of the size 3, where the first value is whether the method succeeded or not, and the next two, the width and height of the image, respectively.
The C decode method also allocated internally, and it returns a buffer of uint8_t values, which we need to copy to the JavaScript side and expose as an Uint8ClampedArray. We need the "clamped" array because the ImageData objects can only work with that, and the data is already clamped :-)
We intercept all GET calls to resources ending in ".webp", fetch the resources, get the array buffer, and the WebPDecoder, decode using an offscreen canvas, and then return a Response based on the blob data. Quite nice.
As I mentioned earlier, we cannot call importScripts from inside a JS file loaded via importScripts so I create a fetchWebPDecoderWithWorkarounds() function, that does that manually as well as blocks until the WASM module is fully loaded.
the web also supports BMP images, which is great because BMP images do not compress their graphical data. Wikipedia also has a great explanation on how the format works, so it was quite straight forward building an encoder:
en.m.wikipedia.org
The resulting encoder is around a mere 60 lines. Very nice!
