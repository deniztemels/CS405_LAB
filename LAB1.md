// CS405 · Lab 1 — your first triangle in WebGPU (starter)
// Work through the TODOs in order. After each one, check the matching
// checkpoint on the lab slides. The reference solution is in ../lab1-solution/.

const canvas = document.querySelector('canvas');

// ---------------------------------------------------------------------------
// TODO 1 — get a device and configure the canvas
//   a) check navigator.gpu exists, throw a clear error if not
//   b) const adapter = await navigator.gpu.requestAdapter()
//   c) const device  = await adapter.requestDevice()
//   d) const ctx     = canvas.getContext('webgpu')
//   e) const format  = navigator.gpu.getPreferredCanvasFormat()
//   f) ctx.configure({ device, format, alphaMode: 'opaque' })
//   g) console.log('WebGPU ready:', format)
// ---------------------------------------------------------------------------

if (!navigator.gpu) throw new Error('WebGPU not supported. Use a recent Chrome');
const adapter = await navigator.gpu.requestAdapter();
if (!adapter) throw new Error('No suitable GPU adapter found.');
const device = await adapter.requestDevice();
const ctx = canvas.getContext('webgpu');
const format = navigator.gpu.getPreferredCanvasFormat();
ctx.configure({ device, format, alphaMode: 'opaque' });
console.log('WebGPU ready:', format);

// ---------------------------------------------------------------------------
// TODO 2 — a shader module and a render pipeline
//   The vertex shader returns clip-space positions for vertex_index 0, 1, 2.
//   The fragment shader returns a solid colour.
//   Then: device.createRenderPipeline({ layout: 'auto', vertex, fragment })
// ---------------------------------------------------------------------------

const shaderCode = /* wgsl */ `
struct Uniforms {
  time:   f32,
  aspect: f32,      // canvas width / height
  mouse:  vec2f,    // mouse position in clip space
};
@group(0) @binding(0) var<uniform> u: Uniforms;

struct VsOut {
  @builtin(position) pos: vec4f,
  @location(0) colour: vec3f,
};

@vertex
fn vs(@builtin(vertex_index) i: u32) -> VsOut {
  // 0..2 = triangle, 3..8 = square (two triangles sharing the diagonal)
  var positions = array<vec2f, 9>(
    vec2f( 0.0,    0.5), vec2f(-0.433, -0.25), vec2f(0.433, -0.25),
    vec2f(-0.3, -0.3), vec2f( 0.3, -0.3), vec2f( 0.3,  0.3),
    vec2f(-0.3, -0.3), vec2f( 0.3,  0.3), vec2f(-0.3,  0.3)
  );
  var colours = array<vec3f, 9>(
    vec3f(1, 0, 0), vec3f(0, 1, 0), vec3f(0, 0, 1),
    vec3f(1, 0, 0), vec3f(0, 1, 0), vec3f(0, 0, 1),
    vec3f(1, 0, 0), vec3f(0, 0, 1), vec3f(1, 1, 0)
  );

  let p = positions[i];
  let a = u.time;
  let R = mat2x2f( cos(a), sin(a),     
                  -sin(a), cos(a));   
  let rotated = R * p;

  // shrink the longer axis so shapes keep their true proportions
  let scale = select(vec2f(1.0, u.aspect), vec2f(1.0 / u.aspect, 1.0), u.aspect > 1.0);

  var out: VsOut;
  out.pos = vec4f(rotated * scale + u.mouse, 0.0, 1.0);
  out.colour = colours[i];
  return out;
}

@fragment
fn fs(in: VsOut) -> @location(0) vec4f {
  return vec4f(in.colour, 1.0);
}
`;

const module = device.createShaderModule({ code: shaderCode });
const pipeline = device.createRenderPipeline({
  layout: 'auto',
  vertex: { module, entryPoint: 'vs' },
  fragment: { module, entryPoint: 'fs', targets: [{ format }] },
  primitive: { topology: 'triangle-list' },
});

// ---------------------------------------------------------------------------
// TODO 3 — a colour per vertex
//   Return a struct from the vertex shader with @location(0) colour,
//   take it as the fragment shader's input, and watch it interpolate.
// ---------------------------------------------------------------------------

// ---------------------------------------------------------------------------
// TODO 4 — a uniform buffer with the time, and rotate the triangle
//   size 16 bytes, usage UNIFORM | COPY_DST
//   bind group from pipeline.getBindGroupLayout(0)
//   device.queue.writeBuffer(...) every frame
// ---------------------------------------------------------------------------

const uniformBuffer = device.createBuffer({
  size: 16, // 4 floats: time, aspect, mouseX, mouseY
  usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});
const bind = device.createBindGroup({
  layout: pipeline.getBindGroupLayout(0),
  entries: [{ binding: 0, resource: { buffer: uniformBuffer } }],
});
const uniformData = new Float32Array(4);

// ---------------------------------------------------------------------------
// TODO 5 — your turn: a square (two triangles), correct aspect ratio,
//   and the shape following the mouse.
// ---------------------------------------------------------------------------

let mouseX = 0;
let mouseY = 0;
canvas.addEventListener('pointermove', (e) => {
  const r = canvas.getBoundingClientRect();
  mouseX = ((e.clientX - r.left) / r.width) * 2 - 1;
  mouseY = -(((e.clientY - r.top) / r.height) * 2 - 1); // clip-space y points up
});

let showSquare = true;
window.addEventListener('keydown', (e) => {
  if (e.code === 'Space') showSquare = !showSquare;
});

const t0 = performance.now();

function resize() {
  const dpr = Math.min(2, window.devicePixelRatio || 1);
  const r = canvas.getBoundingClientRect();
  canvas.width = Math.round(r.width * dpr);
  canvas.height = Math.round(r.height * dpr);
}
window.addEventListener('resize', resize);
resize();

function frame() {
  uniformData[0] = (performance.now() - t0) / 1000;
  uniformData[1] = canvas.width / canvas.height;
  uniformData[2] = mouseX;
  uniformData[3] = mouseY;
  device.queue.writeBuffer(uniformBuffer, 0, uniformData);

  const encoder = device.createCommandEncoder();
  const pass = encoder.beginRenderPass({
    colorAttachments: [{
      view: ctx.getCurrentTexture().createView(),
      clearValue: { r: 0.1, g: 0.1, b: 0.15, a: 1 },
      loadOp: 'clear',
      storeOp: 'store',
    }],
  });
  pass.setPipeline(pipeline);
  pass.setBindGroup(0, bind);
  if (showSquare) pass.draw(6, 1, 3);
  else pass.draw(3, 1, 0); 
  pass.end();
  device.queue.submit([encoder.finish()]);

  requestAnimationFrame(frame);
}
frame();
