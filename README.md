Single Material Mesh Combine for Unity
========================================

A Unity utility that combines multiple child meshes into a single mesh at runtime, reducing draw calls while preserving world-space geometry. Designed for scenarios where many static objects share the same material and need to be batched into one render call.

What It Does
------------

The project provides a runtime mesh combiner that merges all child MeshFilter geometries into one unified mesh on the parent object. After combining, the original child objects are disabled, leaving a single renderer to handle what previously required multiple draw calls.

Why Combine Meshes?
-------------------

### The Problem

When a scene contains many individual mesh objects (e.g., hundreds of cubes, rocks, or tiles), each object typically results in a separate draw call even if they share the same material:

- **High draw call count**: Each MeshRenderer issues at least one draw call. On mobile or low-end platforms, this quickly becomes the primary bottleneck.
- **CPU overhead**: The CPU spends significant time submitting render commands to the GPU.
- **Dynamic batching limits**: Unity's built-in Dynamic Batching has vertex count limits (300 vertices on mobile, 900 on desktop) and does not work well with non-uniform scales or many objects.
- **Static Batching constraints**: Requires objects to be marked static and bakes at build time, which increases build size and memory usage. It also does not work for procedurally placed runtime objects.

### The Solution

Mesh combining merges geometries at the vertex level into a single mesh asset. The result:

| Aspect | Separate Objects | Combined Mesh |
|--------|-----------------|---------------|
| Draw calls | N (one per object) | 1 |
| MeshRenderer count | N | 1 |
| CPU overhead (render thread) | High | Minimal |
| GPU overhead | Same vertex data, multiple state changes | Single contiguous draw |
| Culling granularity | Per-object | Per-combined group |

Trade-offs:
- **Culling**: The entire combined mesh is either rendered or culled as a whole. Individual objects inside the combined volume cannot be frustum-culled separately.
- **Memory**: The combined mesh holds all vertex data in one buffer. This is usually more cache-friendly but removes per-object memory isolation.
- **Dynamic updates**: Once combined, modifying a single child requires recomputing the entire mesh or keeping the original objects active.

### How Unity Handles It Internally

`Mesh.CombineMeshes(CombineInstance[] combine)` takes an array of mesh references and 4x4 transform matrices. It concatenates all vertex buffers (positions, normals, UVs) into a single mesh while applying the provided local-to-world matrices. The resulting mesh is assigned to a single MeshFilter and rendered with one MeshRenderer.

Because the transform matrices are baked into the vertex data during combination, the parent object can then be moved, rotated, or scaled normally without breaking the merged geometry.

Project Structure
-----------------

    Single-Material-Mesh-Combine/
    ├── Assets/
    │   ├── MeshCombiner.cs       # Runtime mesh combining logic
    │   ├── Test.cs               # Spawner: instantiates prefabs and triggers combine
    │   ├── Cube.prefab           # Sample mesh prefab
    │   └── Scenes/               # Unity scene files
    ├── ProjectSettings/          # Unity project settings
    └── README.txt

How It Works
------------

### 1. MeshCombiner.cs

    public class MeshCombiner : MonoBehaviour
    {
        public void CombineMeshes()
        {
            // Cache and reset transform to bake world-space vertices correctly
            Quaternion oldRot = transform.rotation;
            Vector3 oldPos = transform.position;
            transform.rotation = Quaternion.identity;
            transform.position = Vector3.zero;

            // Gather all child MeshFilters
            MeshFilter[] meshFilters = GetComponentsInChildren<MeshFilter>();
            Mesh finalMesh = new Mesh();
            CombineInstance[] combiners = new CombineInstance[meshFilters.Length];

            for (int i = 0; i < meshFilters.Length; i++)
            {
                if (meshFilters[i].transform == transform)
                    continue;

                combiners[i].subMeshIndex = 0;
                combiners[i].mesh = meshFilters[i].sharedMesh;
                combiners[i].transform = meshFilters[i].transform.localToWorldMatrix;
            }

            // Merge into one mesh
            finalMesh.CombineMeshes(combiners);
            GetComponent<MeshFilter>().sharedMesh = finalMesh;

            // Restore parent transform
            transform.rotation = oldRot;
            transform.position = oldPos;

            // Disable original children
            for (int i = 0; i < transform.GetChildCount(); i++)
                transform.GetChild(i).gameObject.SetActive(false);
        }
    }

Key steps:
1. Reset parent rotation and position to identity so `localToWorldMatrix` produces correct world-space vertex positions.
2. Collect every child MeshFilter and build a `CombineInstance` for each, capturing its mesh reference and transform matrix.
3. Call `Mesh.CombineMeshes()` to produce a single unified mesh.
4. Assign the result to the parent's MeshFilter.
5. Restore the parent's original transform.
6. Deactivate child objects so only the combined renderer remains active.

### 2. Test.cs (Spawner & Trigger)

    public class Test : MonoBehaviour
    {
        public int test;                    // Number of objects to spawn
        public GameObject prefab;           // Prefab to instantiate (e.g., Cube)
        public Vector2 minPos, maxPos;      // Random spawn bounds

        void Start()
        {
            for (int i = 0; i < test; i++)
            {
                Instantiate(prefab,
                    new Vector2(Random.RandomRange(minPos.x, maxPos.x),
                                Random.RandomRange(minPos.y, maxPos.y)),
                    Quaternion.identity, this.transform);
            }
        }

        void Update()
        {
            if (Input.GetKeyDown(KeyCode.Space))
                GetComponent<MeshCombiner>().CombineMeshes();
        }
    }

Usage flow:
1. Assign a prefab (e.g., `Cube.prefab`) and set spawn count and bounds in the Inspector.
2. Enter Play Mode: objects spawn as children under the spawner.
3. Press **Space**: all child meshes are merged into one.

Requirements
------------

- Unity 2019.4 LTS or newer
- Built-in Render Pipeline
- All objects to combine must use the **same material** (single submesh)
- Objects need a MeshFilter component

Usage
-----

1. Create an empty GameObject in the scene.
2. Add a `MeshFilter` and `MeshRenderer` to it.
3. Attach `MeshCombiner.cs` to the same object.
4. Attach `Test.cs` and configure:
   - `Prefab`: assign `Cube.prefab` or any mesh prefab
   - `Test`: number of instances to spawn
   - `Min Pos` / `Max Pos`: 2D spawn area
5. Enter Play Mode and press **Space** to combine.

Limitations
-----------

- Designed for **single-material** meshes only (`subMeshIndex = 0`). Multi-material objects require submesh-aware combining.
- Lightmap UVs and baked lighting data on original meshes are not automatically preserved in the combined result.
- Colliders on child objects are disabled but not removed; remove or rebuild collision manually if needed.
- Runtime combination is a one-way operation in this demo. To update, re-enable children and re-run the combine step.

Contact
-------

For questions or suggestions: ysfmerttyldz@gmail.com
