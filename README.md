# Godot Vertex Animation Tool Plugin

A plugin that extends the `MultiMeshInstance3D` node to support instanced vertex animations
using vertex data generated by the 
[Not Unreal Tools - Vertex Animation](https://github.com/yanorax/unreal_tools) Blender add-on, with a vertex shader inside [Godot Engine](https://godotengine.org).

## Preview

https://github.com/user-attachments/assets/49223f09-537d-4572-af84-cab0aa731f85

https://github.com/user-attachments/assets/85120f28-479f-4661-8451-7a7073b20027

## Features

- Can support multiple baked in animations (supports a total of 8192 combined frames).
- Animation tracks' metadata is configured in the editor, not the code.
- Animations tracks can be different frame sizes.
- Ability to set a unique animation track per instance.
- Ability to control the alpha channel for individual instances.
- All the `MultiMeshInstance3D` features such as a unique transform (scale, rotation, and position) per instance.
- Works on all renderers, and on HTML builds.

## Limitations

- Mesh must be less than 8192 vertices.
- Total number of frames for all animations must be less than 8192.
- No blending or transitions between animation tracks possible.
- Current Blender add-on tools tested on Blender 3.3 (Version 4.X not supported)
- `MultiMeshInstance3D` `custom_data` is used by this plugin so you will not have access to it.

## `VATMultiMeshInstance3D`

This plugin provides a new node called `VATMultiMeshInstance3D` which inherits `MultiMeshInstance3D`.

<img src="https://github.com/user-attachments/assets/e979158a-5bb1-43a5-9129-1ca210492b1c" width="503" >

## `VATMultiMeshInstance3D` Properties

- **Instance Count**: `int` = the number of instances
- **Rand Anim Offset**: `bool` =  randomize the animation offset (true/false)
- **Animation Tracks**: `Array[Vector2i]` = the list of animation tracks with start frame = x, end frame = y information. 

<img src="https://github.com/user-attachments/assets/790f897a-ef70-434d-afa3-6acc55c255fc" width="332">

## `VATMultiMeshInstance3D` Functions

### Set/update functions

#### `update_instance_animation_offset(instance_id: int, animation_offset: float)`

Updates the current `instance_id` with the provided `animation_offset` (0..1), unless `rand_anim_offset = false`, where it sets the offset to `0`.

#### `update_instance_track(instance_id: int, track_number: int):`

Updates the current `instance_id` with the provided `track_number` (`0`..`number_of_animation_tracks - 1`)

#### `update_instance_alpha(instance_id: int, alpha: float):`

Updates the current `instance_id` with the provided `alpha` (`0`..`1`)

### Get helper functions

#### `get_start_end_frames_from_track_number(track_number: int) -> Vector2i`

Get animation start/end frame `Vector2i` from `track_number`. `track_number` must be within (`0`..`number_of_animation_tracks - 1`)

#### `get_start_end_frames_from_instance(instance_id: int) -> Vector2i`

Get animation start/end frames `Vector2i` from `instance_id`. Instance must have been initialized.

#### `get_track_number_from_track_vector(track_vector: Vector2i) -> int`

Get `track_number` from start/end frame `Vector2i`. Returns `-1` if not found.

#### `get_track_number_from_instance(instance_id: int) -> int`

Get `track_number` from `instance_id`. Returns `-1` if not found.

## `MutiMeshInstance3D` `custom_data`

`MultiMeshInstance3D` `custom_data` is used by this plugin.  Here is how it is used:

- `custom_data.r` = **animation offset**: used to randomize instances playing the same animation track
- `custom_data.g` = **animation start frame**
- `custom_data.b` = **animation end frame**
- `custom_data.a` = **alpha of mesh**: used to fade in/out a unique instance

## Vertex Animation Shader

The magic of vertex animations happens both in Blender and in the shader. 
This is why you should understand what is happening in the shader.

To make it easy, it is recommended you use `GeometryInstance3D > Geometry > Material Override` 
to add the a new `ShaderMaterial`.

In the `Shader` property select `Quick Load` and select: `vat_multiple_anims.gdshader`

Once loaded expand `Shader Parametrs` and you will have access to configure the following
shader parameters:
	
- `Time Scale`: How quickly animations will play.
- `Offset Map`: A texture that encodes the position of each vertex for every frame.
- `Normal Map`: A texture that encodes the normal of each vertex for every frame.
- `Texture Albedo`: The UV color texture that is used for the mesh.
- `Specular`, `Metallic`, `Roughness`: See Godot [docs](https://docs.godotengine.org/en/stable/tutorials/3d/standard_material_3d.html) for more information.

Make sure both offset and normal textures are imported with Lossless format.

The `custom_data` in the MultiMeshInstance3D and the shader parameters are passed to the shader
to do its magic.  Here is some of the shader code that uses this data:
	
```C++

uniform sampler2D offset_map;
uniform sampler2D normal_map;
uniform sampler2D texture_albedo;

uniform float time_scale = 1.0;

uniform float specular : hint_range(0,1);
uniform float metallic : hint_range(0,1);
uniform float roughness : hint_range(0,1);

varying flat vec4 custom_data;

void vertex(){
	custom_data = INSTANCE_CUSTOM;

	float start_frame = custom_data.g;
	float end_frame = custom_data.b - 1.0;
	
	float time_int = 1.0;
	float time = modf(TIME * time_scale, time_int);
	float num_frames = end_frame - start_frame;
	float frame_offset = num_frames * custom_data.r;
	
	...
	
	
void fragment(){
	vec3 albedo_col = texture(texture_albedo, UV).rgb;

	ALPHA = custom_data.a;  // fader
	ALBEDO = albedo_col.rgb;
```

## Demos

There are two demo scenes in the `demo` subfolder:
	
- **MultipleAnimations**: 90 instances with 5 animations, with different scales, and positions.
- **AlphaTest**: Shows how to control alpha so that you can fade in/out individual instances.

![vat3](https://github.com/user-attachments/assets/84d8c9ee-f343-4226-927b-7bd92e68c5a2)

## Blender 3.3.1 Add-On Guide
1. Download the files from [Not Unreal Tools - Vertex Animation](https://github.com/yanorax/unreal_tools) and install **vertex_animation.py** in the Blender -> **Edit** -> **Preferences...** -> **Add-ons** -> **Install...** menu. In the **3D Viewport** side bar, you should now have a **Not Unreal Tools** menu and if selected it will show a **Vertex Animation** panel.
2. In **Object Mode** select the object you want to process, make sure the current animation you want is selected and playable in the **Timeline**.
3. Adjust the **Frame Start**, **End** and **Step** values as required. Changing these settings will update corresponding **Timeline** values.
4. Click the **Process Anim Meshes** button. This will create a new object named **export_mesh** in the **Outliner**, this is the special mesh that will be animated. In the source .blend file path there will be a newly created folder called **vaexport** and inside will be two files; **normals.png** and **offsets.exr**.
5. The **export_mesh** needs to be exported as a glTF file for importing into Godot. Select the **export_mesh** object in the **Outliner** and then from the Blender **File** menu, select **Export** -> **glTF 2.0 (.glb .gltf)**. Make the following changes to the export options and then click the **Export glTF 2.0** button:
	- Include -> Selected Objects (**enable**)
	- Geometry -> Materials (**disable**)
	- Animation -> Animation, Shape Keys, Skinning (**disable all**)
	- Filename -> can rename to anything

![install](https://github.com/user-attachments/assets/85fd4f4d-177f-48de-bc1c-87c709d924e4)

![tool](https://github.com/user-attachments/assets/a8943e6a-e3cc-447c-ad58-bc5898df2b8f)

## Godot Import Guide
1. You should now have 3 files generated from Blender: **normals.png**, **offsets.exr** and **export_mesh.glb** (whichever filename was chosen, this guide will refer to the default name).
2. Copy the files into the Godot project folder of your choice. Godot will run the import process as soon as it detects the new files. The import settings for each file still need more changes to ensure all of them work properly with the vertex shader.
3. In the Godot **FileSystem** dock, select the glTF file (**export_mesh.glb**) and then click the **Import** dock (default location is docked along side of the **Scene** tree). [Godot Docs - Importing 3D Scenes](https://docs.godotengine.org/en/stable/getting_started/workflow/assets/importing_scenes.html)
4. Make the following adjustments and then click the **Reimport** button. There should be a new file called **export_mesh.mesh** in the same folder as the glTF file (**export_mesh.glb**). 
	- Meshes:
	  - Compress -> (**disable**)
	  - Ensure Tangents -> (**disable**)
	  - Storage -> **Files (.res)**
	- Animation:
	  - Import -> (**disable**)
5. Add a MeshInstance or MultiMeshInstance node to the scene. Drag the **export_mesh.mesh** file into the Mesh parameter slot for a MeshInstance or the Mesh parameter slot inside the MultiMesh for a MultiMeshInstance node. This guide will not cover loading Mesh resources via script.
6. The import settings for **normals.png** and **offsets.exr** will need to be updated after they are added into the shader parameters since Godot will make changes based on what node the image was applied to (3D nodes apply import settings for images used in 3D).
7. Apply the custom vertex animation shader material to a MeshInstance/MultiMeshInstance. Recommend using the GeometryInstance -> Geometry -> Material Override slot.
8. Go to the **Shader Parameters** and click the drop-down arrow and select load for the following parameters:
	- Offset Map -> load **offsets.exr**
	- Normal Map -> load **normals.png**
9. Now find **normals.png** and **offsets.exr** in the **FileSystem** dock, go to Import settings, make the following changes for both files and click the **Reimport** button:
	- Compress:
	  - Mode -> **Lossless** for **normals.png**, **Uncompressed** for **offsets.exr** 
	- Flags:
	  - Repeat -> (**disable**) when changing the current frame using an AnimationPlayer or via script. (**enable**) when looping animations using shader TIME.
	  - Filter -> (**disable**)
	  - Mipmaps -> (**disable**)
10. If you are importing more image files such as albedo textures, refer to [Godot Docs - Importing Images](https://docs.godotengine.org/en/stable/getting_started/workflow/assets/importing_images.html). For palettes and texture masks, recommend using Lossless compression and disable Filter and Mipmaps, so there is no blending of the colours.

## Assets

[Skeleton](https://kaylousberg.itch.io/kaykit-skeletons) by Kay Lousberg - [CC0 License](http://creativecommons.org/publicdomain/zero/1.0/)

[Floor Tile](https://kenney.nl/assets/prototype-textures) by Kenney - [CC0 License](http://creativecommons.org/publicdomain/zero/1.0/)
