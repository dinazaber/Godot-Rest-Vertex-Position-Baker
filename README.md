# Godot Rest Vertex Position Baker

In my game, I rely heavily on shaders that depend on initial vertex positions. A problem occurred when using meshes with a Skeleton3D: bone animations dynamically transform the vertex positions, causing vertex position shaders to behave unexpectedly.
This Godot 4 import script recursively traverses your imported scene and bakes the base rest position of every vertex directly into the CUSTOM1 vertex attribute channel for **each** mesh it finds. You can then reference CUSTOM1.xyz in your shaders to access the original, un-deformed mesh coordinates regardless of skeletal animation.
In addition, this script keeps original UV1 and UV2 maps, vertex colors and the CUSTOM0 slot.

The script:

```gdscript

@tool
extends EditorScenePostImport

func _post_import(scene: Node) -> Object:
	_bake_vertex_positions(scene)
	return scene

func _post_import_custom(scene: Node) -> Object:
	_bake_vertex_positions(scene)
	return scene

func _bake_vertex_positions(node: Node) -> void:
	if node is MeshInstance3D:
		var mi := node as MeshInstance3D
		if mi.mesh:
			var mat = mi.mesh.surface_get_material(0)
			if not mat: mat = mi.get_surface_override_material(0)
			mi.mesh = _bake_mesh(mi.mesh)
			if mat: mi.mesh.surface_set_material(0, mat)
			return

	for child in node.get_children():
		_bake_vertex_positions(child)

func _bake_mesh(src_mesh: Mesh) -> ArrayMesh:
	var out := ArrayMesh.new()
	if src_mesh.get_surface_count() == 0: return out

	var s := 0
	var arrays := src_mesh.surface_get_arrays(s)
	
	# Unpack channels
	var verts: PackedVector3Array = arrays[Mesh.ARRAY_VERTEX]
	var normals: PackedVector3Array = arrays[Mesh.ARRAY_NORMAL]
	var uvs: PackedVector2Array = arrays[Mesh.ARRAY_TEX_UV]
	var colors: PackedColorArray = arrays[Mesh.ARRAY_COLOR]
	var tangents: PackedFloat32Array = arrays[Mesh.ARRAY_TANGENT]
	var bones: PackedInt32Array = arrays[Mesh.ARRAY_BONES]
	var weights: PackedFloat32Array = arrays[Mesh.ARRAY_WEIGHTS]
	var indices: PackedInt32Array = arrays[Mesh.ARRAY_INDEX]

	# READ CUSTOM0 AS FLOATS
	var custom0_floats: PackedFloat32Array = PackedFloat32Array()
	if arrays[Mesh.ARRAY_CUSTOM0]:
		custom0_floats = arrays[Mesh.ARRAY_CUSTOM0]

	var st := SurfaceTool.new()
	st.begin(Mesh.PRIMITIVE_TRIANGLES)
	
	# Read the format layout of the original mesh
	var original_format: int = src_mesh.surface_get_format(s)
	
	# Determine if CUSTOM0 was stored as half-floats or full floats, and preserve it
	if (original_format & (Mesh.ARRAY_CUSTOM_RGBA_HALF << Mesh.ARRAY_FORMAT_CUSTOM0_SHIFT)):
		st.set_custom_format(0, SurfaceTool.CUSTOM_RGBA_HALF)
	else:
		st.set_custom_format(0, SurfaceTool.CUSTOM_RGBA_FLOAT)

	# Explicitly establish our custom float destination for tracking positions in CUSTOM1
	st.set_custom_format(1, SurfaceTool.CUSTOM_RGBA_FLOAT)

	for i in range(verts.size()):
		# Restore pristine normals and tangents for clean texture mappings
		if i < normals.size():
			st.set_normal(normals[i])
		if tangents.size() > 0:
			var t_idx = i * 4
			if t_idx + 3 < tangents.size():
				st.set_tangent(Plane(tangents[t_idx], tangents[t_idx+1], tangents[t_idx+2], tangents[t_idx+3]))
		
		# Restore UV layout mapping
		if i < uvs.size():
			st.set_uv(uvs[i])
			
		# Restore Vertex Colors
		if colors.size() > 0 && i < colors.size():
			st.set_color(colors[i])
		
		# Restore Rigging
		if bones.size() > 0 and weights.size() > 0:
			var b_idx = i * 4
			if b_idx + 3 < bones.size():
				var vertex_bones: PackedInt32Array = [bones[b_idx], bones[b_idx+1], bones[b_idx+2], bones[b_idx+3]]
				var vertex_weights: PackedFloat32Array = [weights[b_idx], weights[b_idx+1], weights[b_idx+2], weights[b_idx+3]]
				st.set_bones(vertex_bones)
				st.set_weights(vertex_weights)
		
		# 5. Reconstruct CUSTOM0 vector data out of the flat float array
		var c0_idx = i * 4
		if custom0_floats.size() > 0 and (c0_idx + 3) < custom0_floats.size():
			st.set_custom(0, Color(
				custom0_floats[c0_idx + 0],
				custom0_floats[c0_idx + 1],
				custom0_floats[c0_idx + 2],
				custom0_floats[c0_idx + 3]
			))
		else:
			st.set_custom(0, Color(0.0, 0.0, 0.0, 0.0))
		
		# Bake unmoving positions into CUSTOM1 channel safely
		var v := verts[i]
		st.set_custom(1, Color(v.x, v.y, v.z, 1.0))
		
		st.add_vertex(v)

	# Build faces safely
	for idx in indices:
		st.add_index(idx)

	out = st.commit()
	return out

```
