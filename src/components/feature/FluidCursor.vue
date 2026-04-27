<script setup>
import { onMounted, onUnmounted, ref } from "vue";

const canvasRef = ref(null);
let animationFrameId = null;

// WGSL Shaders and WebGPU Logic from CodePen
const STRUCT_GRID_SIZE = `
struct GridSize {
  w : f32,
  h : f32,
  dyeW: f32,
  dyeH: f32,
  dx : f32,
  rdx : f32,
  dyeRdx : f32
}`;

const STRUCT_MOUSE = `
struct Mouse {
  pos: vec2<f32>,
  vel: vec2<f32>,
}`;

const COMPUTE_START = `
var pos = vec2<f32>(global_id.xy);
if (pos.x == 0 || pos.y == 0 || pos.x >= uGrid.w - 1 || pos.y >= uGrid.h - 1) {
    return;
}      
let index = ID(pos.x, pos.y);`;

const COMPUTE_START_DYE = `
var pos = vec2<f32>(global_id.xy);
if (pos.x == 0 || pos.y == 0 || pos.x >= uGrid.dyeW - 1 || pos.y >= uGrid.dyeH - 1) {
    return;
}      
let index = ID(pos.x, pos.y);`;

const COMPUTE_START_ALL = `    
var pos = vec2<f32>(global_id.xy);
if (pos.x >= uGrid.w || pos.y >= uGrid.h) {
    return;
}      
let index = ID(pos.x, pos.y);`;

const SPLAT_CODE = `
var m = uMouse.pos;
var v = uMouse.vel*2.;
var splat = createSplat(p, m, v, uRadius);
if (uSymmetry == 1. || uSymmetry == 3.) {splat += createSplat(p, vec2(1. - m.x, m.y), v * vec2(-1., 1.), uRadius);}
if (uSymmetry == 2. || uSymmetry == 3.) {splat += createSplat(p, vec2(m.x, 1. - m.y), v * vec2(1., -1.), uRadius);}
if (uSymmetry == 3. || uSymmetry == 4.) {splat += createSplat(p, vec2(1. - m.x, 1. - m.y), v * vec2(-1., -1.), uRadius);}
`;

const updateVelocityShader = `
${STRUCT_GRID_SIZE}
struct Mouse {
  pos: vec2<f32>,
  vel: vec2<f32>,
}
@group(0) @binding(0) var<storage, read> x_in : array<f32>;
@group(0) @binding(1) var<storage, read> y_in : array<f32>;
@group(0) @binding(2) var<storage, read_write> x_out : array<f32>;
@group(0) @binding(3) var<storage, read_write> y_out : array<f32>;
@group(0) @binding(4) var<uniform> uGrid: GridSize;
@group(0) @binding(5) var<uniform> uMouse: Mouse;
@group(0) @binding(6) var<uniform> uForce : f32;
@group(0) @binding(7) var<uniform> uRadius : f32;
@group(0) @binding(8) var<uniform> uDiffusion : f32;
@group(0) @binding(9) var<uniform> uDt : f32;
@group(0) @binding(10) var<uniform> uTime : f32;
@group(0) @binding(11) var<uniform> uSymmetry : f32;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.w); }
fn createSplat(pos : vec2<f32>, splatPos : vec2<f32>, vel : vec2<f32>, radius : f32) -> vec2<f32> {
  var p = pos - splatPos;
  p.x *= uGrid.w / uGrid.h;
  var v = vel;
  v.x *= uGrid.w / uGrid.h;
  return exp(-dot(p, p) / radius) * v;
}
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
    ${COMPUTE_START}
    var p = pos/vec2(uGrid.w, uGrid.h);
    ${SPLAT_CODE}
    splat *= uForce * uDt * 200. + uTime * 0.0;
    x_out[index] = x_in[index]*uDiffusion + splat.x;
    y_out[index] = y_in[index]*uDiffusion + splat.y;
}`;

const updateDyeShader = `
${STRUCT_GRID_SIZE}
struct Mouse {
  pos: vec2<f32>,
  vel: vec2<f32>,
}
@group(0) @binding(0) var<storage, read> x_in : array<f32>;
@group(0) @binding(1) var<storage, read> y_in : array<f32>;
@group(0) @binding(2) var<storage, read> z_in : array<f32>;
@group(0) @binding(3) var<storage, read_write> x_out : array<f32>;
@group(0) @binding(4) var<storage, read_write> y_out : array<f32>;
@group(0) @binding(5) var<storage, read_write> z_out : array<f32>;
@group(0) @binding(6) var<uniform> uGrid: GridSize;
@group(0) @binding(7) var<uniform> uMouse: Mouse;
@group(0) @binding(8) var<uniform> uForce : f32;
@group(0) @binding(9) var<uniform> uRadius : f32;
@group(0) @binding(10) var<uniform> uDiffusion : f32;
@group(0) @binding(11) var<uniform> uTime : f32;
@group(0) @binding(12) var<uniform> uDt : f32;
@group(0) @binding(13) var<uniform> uSymmetry : f32;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.dyeW); }
fn palette(t : f32, a : vec3<f32>, b : vec3<f32>, c : vec3<f32>, d : vec3<f32> ) -> vec3<f32> {
    return a + b*cos( 6.28318*(c*t+d) );
}
fn createSplat(pos : vec2<f32>, splatPos : vec2<f32>, vel : vec2<f32>, radius : f32) -> vec3<f32> {
  var p = pos - splatPos;
  p.x *= uGrid.w / uGrid.h;
  var v = vel;
  v.x *= uGrid.w / uGrid.h;
  return vec3(exp(-dot(p, p) / radius) * length(v));
}
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
    ${COMPUTE_START_DYE}
    let col_incr = 0.15;
    let col_start = palette(uTime/8., vec3(1), vec3(0.5), vec3(1), vec3(0, col_incr, col_incr*2.));
    var p = pos/vec2(uGrid.dyeW, uGrid.dyeH);
    ${SPLAT_CODE}
    splat *= col_start * uForce * uDt * 100.;
    x_out[index] = max(0., x_in[index]*uDiffusion + splat.x);
    y_out[index] = max(0., y_in[index]*uDiffusion + splat.y);
    z_out[index] = max(0., z_in[index]*uDiffusion + splat.z);
}`;

const advectShader = `
${STRUCT_GRID_SIZE}
@group(0) @binding(0) var<storage, read> x_in : array<f32>;
@group(0) @binding(1) var<storage, read> y_in : array<f32>;
@group(0) @binding(2) var<storage, read> x_vel : array<f32>;
@group(0) @binding(3) var<storage, read> y_vel : array<f32>;
@group(0) @binding(4) var<storage, read_write> x_out : array<f32>;
@group(0) @binding(5) var<storage, read_write> y_out : array<f32>;
@group(0) @binding(6) var<uniform> uGrid : GridSize;
@group(0) @binding(7) var<uniform> uDt : f32;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.w); }
fn in_vec(x : f32, y : f32) -> vec2<f32> { let id = ID(x, y); return vec2(x_in[id], y_in[id]); }
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
    ${COMPUTE_START}
    var x = pos.x - uDt * uGrid.rdx * x_vel[index];
    var y = pos.y - uDt * uGrid.rdx * y_vel[index];
    x = clamp(x, 0.0, uGrid.w - 1.0);
    y = clamp(y, 0.0, uGrid.h - 1.0);
    let x1 = floor(x);
    let y1 = floor(y);
    let x2 = x1 + 1;
    let y2 = y1 + 1;
    let TL = in_vec(x1, y2);
    let TR = in_vec(x2, y2);
    let BL = in_vec(x1, y1);
    let BR = in_vec(x2, y1);
    let bilerp = mix( mix(BL, BR, fract(x)), mix(TL, TR, fract(x)), fract(y) );
    x_out[index] = bilerp.x;
    y_out[index] = bilerp.y;
}`;

const advectDyeShader = `
${STRUCT_GRID_SIZE}
@group(0) @binding(0) var<storage, read> x_in : array<f32>;
@group(0) @binding(1) var<storage, read> y_in : array<f32>;
@group(0) @binding(2) var<storage, read> z_in : array<f32>;
@group(0) @binding(3) var<storage, read> x_vel : array<f32>;
@group(0) @binding(4) var<storage, read> y_vel : array<f32>;
@group(0) @binding(5) var<storage, read_write> x_out : array<f32>;
@group(0) @binding(6) var<storage, read_write> y_out : array<f32>;
@group(0) @binding(7) var<storage, read_write> z_out : array<f32>;
@group(0) @binding(8) var<uniform> uGrid : GridSize;
@group(0) @binding(9) var<uniform> uDt : f32;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.dyeW); }
fn in_vec3(x : f32, y : f32) -> vec3<f32> { let id = ID(x, y); return vec3(x_in[id], y_in[id], z_in[id]); }
fn vel(x : f32, y : f32) -> vec2<f32> { 
  let id = u32(i32(x) + i32(y) * i32(uGrid.w));
  return vec2(x_vel[id], y_vel[id]);
}
fn vel_bilerp(x0 : f32, y0 : f32) -> vec2<f32> {
    var x = x0 * uGrid.w / uGrid.dyeW;
    var y = y0 * uGrid.h / uGrid.dyeH;
    x = clamp(x, 0.0, uGrid.w - 1.0);
    y = clamp(y, 0.0, uGrid.h - 1.0);
    let x1 = floor(x);
    let y1 = floor(y);
    let x2 = x1 + 1;
    let y2 = y1 + 1;
    let TL = vel(x1, y2);
    let TR = vel(x2, y2);
    let BL = vel(x1, y1);
    let BR = vel(x2, y1);
    return mix( mix(BL, BR, fract(x)), mix(TL, TR, fract(x)), fract(y) );
}
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
    ${COMPUTE_START_DYE}
    let V = vel_bilerp(pos.x, pos.y);
    var x = pos.x - uDt * uGrid.dyeRdx * V.x;
    var y = pos.y - uDt * uGrid.dyeRdx * V.y;
    x = clamp(x, 0.0, uGrid.dyeW - 1.0);
    y = clamp(y, 0.0, uGrid.dyeH - 1.0);
    let x1 = floor(x);
    let y1 = floor(y);
    let x2 = x1 + 1;
    let y2 = y1 + 1;
    let TL = in_vec3(x1, y2);
    let TR = in_vec3(x2, y2);
    let BL = in_vec3(x1, y1);
    let BR = in_vec3(x2, y1);
    let bilerp = mix( mix(BL, BR, fract(x)), mix(TL, TR, fract(x)), fract(y) );
    x_out[index] = bilerp.x;
    y_out[index] = bilerp.y;
    z_out[index] = bilerp.z;
}`;

const divergenceShader = `   
${STRUCT_GRID_SIZE}
@group(0) @binding(0) var<storage, read> x_vel : array<f32>;
@group(0) @binding(1) var<storage, read> y_vel : array<f32>;
@group(0) @binding(2) var<storage, read_write> div : array<f32>;
@group(0) @binding(3) var<uniform> uGrid : GridSize;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.w); }
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
  ${COMPUTE_START}
  let L = x_vel[ID(pos.x - 1, pos.y)];
  let R = x_vel[ID(pos.x + 1, pos.y)];
  let B = y_vel[ID(pos.x, pos.y - 1)];
  let T = y_vel[ID(pos.x, pos.y + 1)];
  div[index] = 0.5 * uGrid.rdx * ((R - L) + (T - B));
}`;

const pressureShader = `      
${STRUCT_GRID_SIZE}
@group(0) @binding(0) var<storage, read> pres_in : array<f32>;
@group(0) @binding(1) var<storage, read> div : array<f32>;
@group(0) @binding(2) var<storage, read_write> pres_out : array<f32>;
@group(0) @binding(3) var<uniform> uGrid : GridSize;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.w); }
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
  ${COMPUTE_START}
  let Lx = pres_in[ID(pos.x - 1, pos.y)];
  let Rx = pres_in[ID(pos.x + 1, pos.y)];
  let Bx = pres_in[ID(pos.x, pos.y - 1)];
  let Tx = pres_in[ID(pos.x, pos.y + 1)];
  pres_out[index] = (Lx + Rx + Bx + Tx - (uGrid.dx * uGrid.dx) * div[index]) * .25;
}`;

const gradientSubtractShader = `      
${STRUCT_GRID_SIZE}
@group(0) @binding(0) var<storage, read> pressure : array<f32>;
@group(0) @binding(1) var<storage, read> x_vel : array<f32>;
@group(0) @binding(2) var<storage, read> y_vel : array<f32>;
@group(0) @binding(3) var<storage, read_write> x_out : array<f32>;
@group(0) @binding(4) var<storage, read_write> y_out : array<f32>;
@group(0) @binding(5) var<uniform> uGrid : GridSize;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.w); }
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
  ${COMPUTE_START}
  let xL = pressure[ID(pos.x - 1, pos.y)];
  let xR = pressure[ID(pos.x + 1, pos.y)];
  let yB = pressure[ID(pos.x, pos.y - 1)];
  let yT = pressure[ID(pos.x, pos.y + 1)];
  x_out[index] = x_vel[index] - .5 * uGrid.rdx * (xR - xL);
  y_out[index] = y_vel[index] - .5 * uGrid.rdx * (yT - yB);
}`;

const vorticityShader = `      
${STRUCT_GRID_SIZE}
@group(0) @binding(0) var<storage, read> x_vel : array<f32>;
@group(0) @binding(1) var<storage, read> y_vel : array<f32>;
@group(0) @binding(2) var<storage, read_write> vorticity : array<f32>;
@group(0) @binding(3) var<uniform> uGrid : GridSize;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.w); }
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
  ${COMPUTE_START}
  let Ly = y_vel[ID(pos.x - 1, pos.y)];
  let Ry = y_vel[ID(pos.x + 1, pos.y)];
  let Bx = x_vel[ID(pos.x, pos.y - 1)];
  let Tx = x_vel[ID(pos.x, pos.y + 1)];
  vorticity[index] = 0.5 * uGrid.rdx * ((Ry - Ly) - (Tx - Bx));
}`;

const vorticityConfinmentShader = `      
${STRUCT_GRID_SIZE}
@group(0) @binding(0) var<storage, read> x_vel_in : array<f32>;
@group(0) @binding(1) var<storage, read> y_vel_in : array<f32>;
@group(0) @binding(2) var<storage, read> vorticity : array<f32>;
@group(0) @binding(3) var<storage, read_write> x_vel_out : array<f32>;
@group(0) @binding(4) var<storage, read_write> y_vel_out : array<f32>;
@group(0) @binding(5) var<uniform> uGrid : GridSize;
@group(0) @binding(6) var<uniform> uDt : f32;
@group(0) @binding(7) var<uniform> uVorticity : f32;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.w); }
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
  ${COMPUTE_START}
  let L = abs(vorticity[ID(pos.x - 1, pos.y)]);
  let R = abs(vorticity[ID(pos.x + 1, pos.y)]);
  let B = abs(vorticity[ID(pos.x, pos.y - 1)]);
  let T = abs(vorticity[ID(pos.x, pos.y + 1)]);
  var force = 0.5 * uGrid.rdx * vec2(T - B, R - L);
  let magSqr = max(2.4414e-4, dot(force, force));
  force = force / sqrt(magSqr);
  force *= uGrid.dx * uVorticity * uDt * vorticity[index] * vec2(1, -1);
  x_vel_out[index] = x_vel_in[index] + force.x;
  y_vel_out[index] = y_vel_in[index] + force.y;
}`;

const clearShader = `  
${STRUCT_GRID_SIZE}
@group(0) @binding(0) var<storage, read> x_in : array<f32>;
@group(0) @binding(1) var<storage, read_write> x_out : array<f32>;
@group(0) @binding(2) var<uniform> uGrid : GridSize;
@group(0) @binding(3) var<uniform> uScale : f32;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.w); }
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
  ${COMPUTE_START_ALL}
  x_out[index] = x_in[index] * uScale;
}`;

const boundaryShader = `
${STRUCT_GRID_SIZE}
@group(0) @binding(0) var<storage, read> x_in : array<f32>;
@group(0) @binding(1) var<storage, read> y_in : array<f32>;
@group(0) @binding(2) var<storage, read_write> x_out : array<f32>;
@group(0) @binding(3) var<storage, read_write> y_out : array<f32>;
@group(0) @binding(4) var<uniform> uGrid : GridSize;
@group(0) @binding(5) var<uniform> containFluid : f32;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.w); }
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
  ${COMPUTE_START_ALL}
  var scaleX = 1.;
  var scaleY = 1.;
  var targetPos = pos;
  if (pos.x == 0) { targetPos.x += 1; scaleX = -1.; }
  else if (pos.x == uGrid.w - 1) { targetPos.x -= 1; scaleX = -1.; }
  if (pos.y == 0) { targetPos.y += 1; scaleY = -1.; }
  else if (pos.y == uGrid.h - 1) { targetPos.y -= 1; scaleY = -1.; }
  if (containFluid == 0.) { scaleX = 1.; scaleY = 1.; }
  let tid = ID(targetPos.x, targetPos.y);
  x_out[index] = x_in[tid] * scaleX;
  y_out[index] = y_in[tid] * scaleY;
}`;

const boundaryValueShader = `    
${STRUCT_GRID_SIZE}
@group(0) @binding(0) var<storage, read> x_in : array<f32>;
@group(0) @binding(1) var<storage, read_write> x_out : array<f32>;
@group(0) @binding(2) var<uniform> uGrid : GridSize;
fn ID(x : f32, y : f32) -> u32 { return u32(x + y * uGrid.w); }
@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
  ${COMPUTE_START_ALL}
  var targetPos = pos;
  if (pos.x == 0) { targetPos.x += 1; }
  else if (pos.x == uGrid.w - 1) { targetPos.x -= 1; }
  if (pos.y == 0) { targetPos.y += 1; }
  else if (pos.y == uGrid.h - 1) { targetPos.y -= 1; }
  x_out[index] = x_in[ID(targetPos.x, targetPos.y)];
}`;

const renderShader = `
${STRUCT_GRID_SIZE}
struct VertexOut {
  @builtin(position) position : vec4<f32>,
  @location(1) uv : vec2<f32>,
};
@group(0) @binding(0) var<storage, read> fieldX : array<f32>;
@group(0) @binding(1) var<storage, read> fieldY : array<f32>;
@group(0) @binding(2) var<storage, read> fieldZ : array<f32>;
@group(0) @binding(3) var<uniform> uGrid : GridSize;
@group(0) @binding(4) var<uniform> multiplier : f32;
@vertex
fn vertex_main(@location(0) position: vec4<f32>) -> VertexOut {
    var output : VertexOut;
    output.position = position;
    output.uv = position.xy*.5+.5;
    return output;
}
@fragment
fn fragment_main(fragData : VertexOut) -> @location(0) vec4<f32> {
    let fuv = floor(fragData.uv * vec2(uGrid.dyeW, uGrid.dyeH));
    let id = u32(fuv.x + fuv.y * uGrid.dyeW);
    let col = vec3(fieldX[id], fieldY[id], fieldZ[id]);
    return vec4(col * multiplier, clamp(length(col), 0.0, 1.0));
}
`;

// Helper classes for WebGPU
class DynamicBuffer {
  constructor(device, { dims = 1, w, h }) {
    this.dims = dims;
    this.bufferSize = w * h * 4;
    this.w = w;
    this.h = h;
    this.buffers = Array.from({ length: dims }, () =>
      device.createBuffer({
        size: this.bufferSize,
        usage:
          GPUBufferUsage.STORAGE |
          GPUBufferUsage.COPY_SRC |
          GPUBufferUsage.COPY_DST,
      })
    );
  }
  copyTo(buffer, commandEncoder) {
    for (let i = 0; i < Math.max(this.dims, buffer.dims); i++) {
      commandEncoder.copyBufferToBuffer(
        this.buffers[Math.min(i, this.buffers.length - 1)],
        0,
        buffer.buffers[Math.min(i, buffer.buffers.length - 1)],
        0,
        this.bufferSize
      );
    }
  }
  clear(device) {
    const empty = new Float32Array(this.w * this.h);
    this.buffers.forEach((b) => device.queue.writeBuffer(b, 0, empty));
  }
}

class Uniform {
  constructor(device, name, { size, value }) {
    this.name = name;
    this.size = size || (Array.isArray(value) ? value.length : 1);
    this.buffer = device.createBuffer({
      size: this.size * 4,
      usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
    });
    if (value !== undefined) this.update(device, value);
  }
  update(device, value) {
    const arr = Array.isArray(value)
      ? new Float32Array(value)
      : new Float32Array([value]);
    device.queue.writeBuffer(this.buffer, 0, arr);
  }
}

class Program {
  constructor(device, { buffers, uniforms, shader, format }) {
    const module = device.createShaderModule({ code: shader });
    this.pipeline = device.createComputePipeline({
      layout: "auto",
      compute: { module, entryPoint: "main" },
    });
    const entries = [
      ...buffers.flatMap((b) => b.buffers),
      ...uniforms.map((u) => u.buffer),
    ].map((resource, i) => ({
      binding: i,
      resource: { buffer: resource },
    }));
    this.bindGroup = device.createBindGroup({
      layout: this.pipeline.getBindGroupLayout(0),
      entries,
    });
  }
  dispatch(passEncoder, x, y) {
    passEncoder.setPipeline(this.pipeline);
    passEncoder.setBindGroup(0, this.bindGroup);
    passEncoder.dispatchWorkgroups(Math.ceil(x / 8), Math.ceil(y / 8));
  }
}

const settings = {
  grid_size: 64,
  dye_size: 256,
  sim_speed: 5,
  contain_fluid: true,
  vel_force: 0.28,
  vel_radius: 0.001,
  vel_diff: 1.0,
  dye_force: 0.8,
  dye_radius: 0.0035,
  dye_diff: 0.962,
  vorticity: 0.0,
  pressure_iterations: 8,
};

onMounted(async () => {
  if (!navigator.gpu) return;
  const adapter = await navigator.gpu.requestAdapter();
  const device = await adapter.requestDevice();
  const canvas = canvasRef.value;
  const context = canvas.getContext("webgpu");
  const format = navigator.gpu.getPreferredCanvasFormat();
  context.configure({ device, format, alphaMode: "premultiplied" });

  const rect = canvas.getBoundingClientRect();
  const dpr = window.devicePixelRatio || 1;
  const aspect = rect.width / rect.height;

  const getDims = (base) => {
    let w = aspect > 1 ? Math.floor(base * aspect) : base;
    let h = aspect > 1 ? base : Math.floor(base / aspect);
    return { w: Math.floor(w * dpr), h: Math.floor(h * dpr) };
  };

  const sim = getDims(settings.grid_size);
  const dyeSize = getDims(settings.dye_size);
  canvas.width = dyeSize.w;
  canvas.height = dyeSize.h;

  const dx = 1 / (settings.grid_size * 4);
  const rdx = settings.grid_size * 4;
  const dyeRdx = settings.dye_size * 4;

  const vel0 = new DynamicBuffer(device, { dims: 2, w: sim.w, h: sim.h });
  const vel1 = new DynamicBuffer(device, { dims: 2, w: sim.w, h: sim.h });
  const dye0 = new DynamicBuffer(device, {
    dims: 3,
    w: dyeSize.w,
    h: dyeSize.h,
  });
  const dye1 = new DynamicBuffer(device, {
    dims: 3,
    w: dyeSize.w,
    h: dyeSize.h,
  });
  const div = new DynamicBuffer(device, { dims: 1, w: sim.w, h: sim.h });
  const divB = new DynamicBuffer(device, { dims: 1, w: sim.w, h: sim.h });
  const pres0 = new DynamicBuffer(device, { dims: 1, w: sim.w, h: sim.h });
  const pres1 = new DynamicBuffer(device, { dims: 1, w: sim.w, h: sim.h });
  const vort = new DynamicBuffer(device, { dims: 1, w: sim.w, h: sim.h });

  const uGrid = new Uniform(device, "grid", {
    value: [sim.w, sim.h, dyeSize.w, dyeSize.h, dx, rdx, dyeRdx],
  });
  const uMouse = new Uniform(device, "mouse", { size: 4, value: [0, 0, 0, 0] });
  const uDt = new Uniform(device, "dt", { value: 0 });
  const uTime = new Uniform(device, "time", { value: 0 });
  const uSymmetry = new Uniform(device, "symmetry", { value: 0 });

  const progVel = new Program(device, {
    buffers: [vel0, vel1],
    uniforms: [
      uGrid,
      uMouse,
      new Uniform(device, "", { value: settings.vel_force }),
      new Uniform(device, "", { value: settings.vel_radius }),
      new Uniform(device, "", { value: settings.vel_diff }),
      uDt,
      uTime,
      uSymmetry,
    ],
    shader: updateVelocityShader,
  });
  const progDye = new Program(device, {
    buffers: [dye0, dye1],
    uniforms: [
      uGrid,
      uMouse,
      new Uniform(device, "", { value: settings.dye_force }),
      new Uniform(device, "", { value: settings.dye_radius }),
      new Uniform(device, "", { value: settings.dye_diff }),
      uTime,
      uDt,
      uSymmetry,
    ],
    shader: updateDyeShader,
  });
  const progAdvVel = new Program(device, {
    buffers: [vel1, vel1, vel0],
    uniforms: [uGrid, uDt],
    shader: advectShader,
  });
  const progBoundVel = new Program(device, {
    buffers: [vel0, vel1],
    uniforms: [
      uGrid,
      new Uniform(device, "", { value: settings.contain_fluid ? 1 : 0 }),
    ],
    shader: boundaryShader,
  });
  const progDiv = new Program(device, {
    buffers: [vel1, div],
    uniforms: [uGrid],
    shader: divergenceShader,
  });
  const progBoundDiv = new Program(device, {
    buffers: [div, divB],
    uniforms: [uGrid],
    shader: boundaryValueShader,
  });
  const progPres = new Program(device, {
    buffers: [pres0, divB, pres1],
    uniforms: [uGrid],
    shader: pressureShader,
  });
  const progBoundPres = new Program(device, {
    buffers: [pres1, pres0],
    uniforms: [uGrid],
    shader: boundaryValueShader,
  });
  const progGrad = new Program(device, {
    buffers: [pres0, vel1, vel0],
    uniforms: [uGrid],
    shader: gradientSubtractShader,
  });
  const progClearPres = new Program(device, {
    buffers: [pres0, pres1],
    uniforms: [uGrid, new Uniform(device, "", { value: 0 })],
    shader: clearShader,
  });
  const progVort = new Program(device, {
    buffers: [vel0, vort],
    uniforms: [uGrid],
    shader: vorticityShader,
  });
  const progVortConf = new Program(device, {
    buffers: [vel0, vort, vel1],
    uniforms: [
      uGrid,
      uDt,
      new Uniform(device, "", { value: settings.vorticity }),
    ],
    shader: vorticityConfinmentShader,
  });
  const progAdvDye = new Program(device, {
    buffers: [dye1, vel1, dye0],
    uniforms: [uGrid, uDt],
    shader: advectDyeShader,
  });

  // Render Pipeline
  const renderModule = device.createShaderModule({ code: renderShader });
  const renderPipeline = device.createRenderPipeline({
    layout: "auto",
    vertex: {
      module: renderModule,
      entryPoint: "vertex_main",
      buffers: [
        {
          attributes: [{ shaderLocation: 0, offset: 0, format: "float32x4" }],
          arrayStride: 16,
        },
      ],
    },
    fragment: {
      module: renderModule,
      entryPoint: "fragment_main",
      targets: [{ format }],
    },
    primitive: { topology: "triangle-list" },
  });
  const vBuf = device.createBuffer({
    size: 96,
    usage: GPUBufferUsage.VERTEX,
    mappedAtCreation: true,
  });
  new Float32Array(vBuf.getMappedRange()).set([
    -1, -1, 0, 1, 1, -1, 0, 1, -1, 1, 0, 1, -1, 1, 0, 1, 1, -1, 0, 1, 1, 1, 0,
    1,
  ]);
  vBuf.unmap();
  const renderBind = device.createBindGroup({
    layout: renderPipeline.getBindGroupLayout(0),
    entries: [
      ...dye0.buffers.map((b, i) => ({ binding: i, resource: { buffer: b } })),
      { binding: 3, resource: { buffer: uGrid.buffer } },
      {
        binding: 4,
        resource: { buffer: new Uniform(device, "", { value: 1 }).buffer },
      },
    ],
  });

  let mx = 0,
    my = 0,
    lmx = 0,
    lmy = 0,
    time = 0,
    last = performance.now();
  const onMove = (e) => {
    const p = e.touches ? e.touches[0] : e;
    mx = (p.clientX - rect.left) / rect.width;
    my = 1 - (p.clientY - rect.top) / rect.height;
  };
  window.addEventListener("mousemove", onMove);
  window.addEventListener("touchstart", onMove);
  window.addEventListener("touchmove", onMove);

  const frame = () => {
    const now = performance.now();
    const dt = Math.min(1 / 60, (now - last) / 1000) * settings.sim_speed;
    time += dt;
    last = now;
    uDt.update(device, dt);
    uTime.update(device, time);
    uMouse.update(device, [mx, my, mx - lmx, my - lmy]);
    lmx = mx;
    lmy = my;

    const encoder = device.createCommandEncoder();
    const pass = encoder.beginComputePass();
    progDye.dispatch(pass, dyeSize.w, dyeSize.h);
    progVel.dispatch(pass, sim.w, sim.h);
    progAdvVel.dispatch(pass, sim.w, sim.h);
    progBoundVel.dispatch(pass, sim.w, sim.h);
    progDiv.dispatch(pass, sim.w, sim.h);
    progBoundDiv.dispatch(pass, sim.w, sim.h);
    for (let i = 0; i < settings.pressure_iterations; i++) {
      progPres.dispatch(pass, sim.w, sim.h);
      progBoundPres.dispatch(pass, sim.w, sim.h);
    }
    progGrad.dispatch(pass, sim.w, sim.h);
    progClearPres.dispatch(pass, sim.w, sim.h);
    progVort.dispatch(pass, sim.w, sim.h);
    progVortConf.dispatch(pass, sim.w, sim.h);
    progAdvDye.dispatch(pass, dyeSize.w, dyeSize.h);
    pass.end();

    dye0.copyTo(dye1, encoder);
    vel1.copyTo(vel0, encoder);
    pres1.copyTo(pres0, encoder);

    const rPass = encoder.beginRenderPass({
      colorAttachments: [
        {
          view: context.getCurrentTexture().createView(),
          clearValue: { r: 0, g: 0, b: 0, a: 0 },
          loadOp: "clear",
          storeOp: "store",
        },
      ],
    });
    rPass.setPipeline(renderPipeline);
    rPass.setBindGroup(0, renderBind);
    rPass.setVertexBuffer(0, vBuf);
    rPass.draw(6);
    rPass.end();
    device.queue.submit([encoder.finish()]);
    animationFrameId = requestAnimationFrame(frame);
  };
  frame();
});

onUnmounted(() => {
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
  window.removeEventListener("mousemove", () => {});
  window.removeEventListener("touchstart", () => {});
  window.removeEventListener("touchmove", () => {});
});
</script>

<template>
  <canvas ref="canvasRef" class="fluid-cursor-canvas" />
</template>

<style>
/* Global reset to remove padding/margin that might cause gaps */
html,
body,
.v-application {
  margin: 0 !important;
  padding: 0 !important;
}

#app {
  padding: 0 !important;
  margin: 0 !important;
}
</style>

<style scoped>
.fluid-cursor-canvas {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  pointer-events: none;
  z-index: 9999;
  mix-blend-mode: screen;
}
</style>
