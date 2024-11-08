---
layout: post
title: SDF Rendering in Godot - Scene Data
tags:
- Godot
- Projects
- SDF
---

I have been working on a small project to figure out solutions to real time SDF rendering in Godot. Obviously there are solutions floating around online for just creating a simple SDF rendering shader inside of gdshader. However, I was looking for a more robust, general solution that included editor integration and the new  <a href="https://docs.godotengine.org/en/stable/tutorials/rendering/compositor.html">compositor system</a>. There are a few advantages to using the compositor, mostly the benefits that come from using GLSL instead of the GDShader language. GDShader is very powerful, but unfortunately the features it lacks are especially important for SDF rendering. The compositor system uses Godot's solution for <a href="https://docs.godotengine.org/en/stable/tutorials/shaders/compute_shaders.html#">compute shaders</a>, namely, <i>native</i> GLSL. Allowing an interesting alternative. 

While this dodges some problems it introduces its own and requires sometimes mountains of boilerplate code. One huge problem you will quickly encounter is compute shaders don't have an easy way to access important information from the Godot scene. GDScript includes tons of <a href="https://docs.godotengine.org/en/stable/tutorials/shaders/shader_reference/shading_language.html#built-in-variables">built-ins</a> depending on the type of shader being created and all of them are missing in compute shaders. Eventually, I found a <a href="https://github.com/godotengine/godot/pull/80214#issuecomment-1953258434">Github issue</a> talking about this problem and potential solutions. 

Essentially, we can pass a uniform buffer object into our shader containing the normal scene information and handle the object on the GLSL side to achieve the same results. The best example of this I have seen has been the <a href="https://github.com/pink-arcana/godot-distance-field-outlines/tree/main">Distance Field Outlines</a> project by Pink Arcana.

We need to grab the built-in SceneData UBO in the compositor effect and pass it into our shader.

```
func get_scene_data_ubo() -> RDUniform:
	if not render_scene_data:
		return null

	var scene_data_buffer : RID = render_scene_data.get_uniform_buffer()
	var scene_data_buffer_uniform := RDUniform.new()
	scene_data_buffer_uniform.uniform_type = RenderingDevice.UNIFORM_TYPE_UNIFORM_BUFFER
	scene_data_buffer_uniform.binding = SCENE_DATA_BINDING
	scene_data_buffer_uniform.add_id(scene_data_buffer)
	return scene_data_buffer_uniform
```
```
# Scene Data Uniform Buffer
var scene_uniform := RDUniform.new()
scene_uniform.uniform_type = RenderingDevice.UNIFORM_TYPE_UNIFORM_BUFFER
scene_uniform.binding = 0
scene_uniform.add_id(scene_data_buffer)
var scene_data_uniform_set := UniformSetCacheRD.get_cache(shader, 0,[scene_uniform])

var compute_list := rd.compute_list_begin()
rd.compute_list_bind_compute_pipeline(compute_list, pipeline)
rd.compute_list_bind_uniform_set(compute_list, scene_data_uniform_set, 0)
rd.compute_list_dispatch(compute_list, x_groups, y_groups, z_groups)
rd.compute_list_end()
```

Now, on the GLSL side we only need a way to structure the UBO. I used an <a href="https://github.com/pink-arcana/godot-distance-field-outlines/blob/main/project/df_outline_ce/shaders/includes/scene_data.glsl">include file</a> that pink-arcana provided. The structure is fixed so be sure to match the struct exactly.

```
struct SceneData {
	highp mat4 projection_matrix;
	highp mat4 inv_projection_matrix;
	highp mat4 inv_view_matrix;
	highp mat4 view_matrix;

	// only used for multiview
	highp mat4 projection_matrix_view[2];
	highp mat4 inv_projection_matrix_view[2];
	highp vec4 eye_offset[2];

	// Used for billboards to cast correct shadows.
	highp mat4 main_cam_inv_view_matrix;

	highp vec2 viewport_size;
	highp vec2 screen_pixel_size;

	// Use vec4s because std140 doesn't play nice with vec2s, z and w are wasted.
	highp vec4 directional_penumbra_shadow_kernel[32];
	highp vec4 directional_soft_shadow_kernel[32];
	highp vec4 penumbra_shadow_kernel[32];
	highp vec4 soft_shadow_kernel[32];

	mediump mat3 radiance_inverse_xform;

	mediump vec4 ambient_light_color_energy;

	mediump float ambient_color_sky_mix;
	bool use_ambient_light;
	bool use_ambient_cubemap;
	bool use_reflection_cubemap;

	highp vec2 shadow_atlas_pixel_size;
	highp vec2 directional_shadow_pixel_size;

	uint directional_light_count;
	mediump float dual_paraboloid_side;
	highp float z_far;
	highp float z_near;

	bool roughness_limiter_enabled;
	mediump float roughness_limiter_amount;
	mediump float roughness_limiter_limit;
	mediump float opaque_prepass_threshold;

	bool fog_enabled;
	uint fog_mode;
	highp float fog_density;
	highp float fog_height;
	highp float fog_height_density;

	highp float fog_depth_curve;
	highp float pad;
	highp float fog_depth_begin;

	mediump vec3 fog_light_color;
	highp float fog_depth_end;

	mediump float fog_sun_scatter;
	mediump float fog_aerial_perspective;
	highp float time;
	mediump float reflection_multiplier; // one normally, zero when rendering reflections

	vec2 taa_jitter;
	bool material_uv2_mode;
	float emissive_exposure_normalization;

	float IBL_exposure_normalization;
	bool pancake_shadows;
	uint camera_visible_layers;
	float pass_alpha_multiplier;
};
```

<i>Voila!</i> Your scene data is in the compositor. I needed this to preform basic projection camera-space and screen-space projections to begin raymarching.

```
// Assuming uv is already calculated
highp vec2 ndc = uv * 2.0 - 1.0;  // Map UV to NDC [-1, 1]
highp vec4 clip_space = vec4(ndc, 1.0, 1.0);  // NDC to clip space
highp vec4 view_space_ray = scene_data.inv_projection_matrix * clip_space;
view_space_ray /= view_space_ray.w;
highp vec3 world_space_ray = (scene_data.inv_view_matrix * vec4(view_space_ray.xyz, 0.0)).xyz;

// Normalize the ray direction in world space
highp vec3 ray_dir = normalize(world_space_ray.xyz);

// Get the camera's world space position
highp vec3 camera_position = scene_data.inv_view_matrix[3].xyz;


// Prevent reading/writing out of bounds.
if (uv.x >= size.x || uv.y >= size.y) {
    return;
}

// Prevent reading/writing out of bounds.
if (abs(ndc.x) >= 1 || abs(ndc.y) >= 1) {
    return;
}

vec4 rayout = raymarchDefaultRender(camera_position, ray_dir);
```

<p><img src="{{ site.baseurl }}static/img/sdf1.png" alt="Test Scene!" class="bigimage" /></p>
