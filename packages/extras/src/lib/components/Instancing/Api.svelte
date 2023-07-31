<script lang="ts">
  import { T, useFrame } from '@threlte/core'
  import type { InstancedMesh } from 'three'
  import { DynamicDrawUsage, Matrix4, Quaternion, Vector3 } from 'three'
  import { createApi } from './api'

  export let instancedMesh: InstancedMesh
  export let id: string
  export let limit: number
  export let range: number
  export let update: boolean

  const { instances } = createApi(instancedMesh, id)

  const tempMatrix = new Matrix4()

  const matrices = new Float32Array(limit * 16)
  for (let i = 0; i < limit; i++) tempMatrix.identity().toArray(matrices, i * 16)

  const colors = new Float32Array([...new Array(limit * 3)].map(() => 1))

  const parentMatrix = new Matrix4()
  const instanceMatrix = new Matrix4()
  const translation = new Vector3()
  const rotation = new Quaternion()
  const scale = new Vector3()

  let initialUpdateDone = false
  useFrame(() => {
    instancedMesh.updateMatrix()
    if (update || !initialUpdateDone) {
      instancedMesh.updateMatrixWorld()
      parentMatrix.copy(instancedMesh.matrixWorld).invert()

      const updateRange = Math.min(limit, range !== undefined ? range : limit, $instances.length)
      instancedMesh.count = updateRange
      instancedMesh.instanceMatrix.updateRange.count = updateRange * 16
      if (instancedMesh.instanceColor) {
        instancedMesh.instanceColor.updateRange.count = updateRange * 3
      }

      // update the transform matrices and colors
      if (instancedMesh.instanceColor) {
        instancedMesh.instanceColor.needsUpdate = true
      }
      instancedMesh.instanceMatrix.needsUpdate = true

      for (let i = 0; i < instances.current.length; i++) {
        const instance = instances.current[i]
        // Multiply the inverse of the InstancedMesh world matrix or else
        // Instances will be double-transformed if <Instances> isn't at identity
        instance.matrixWorld.decompose(translation, rotation, scale)
        instanceMatrix.compose(translation, rotation, scale).premultiply(parentMatrix)
        instanceMatrix.toArray(matrices, i * 16)
        instance.color.toArray(colors, i * 3)
      }

      initialUpdateDone = true
    }
  })
</script>

<T.InstancedBufferAttribute
  attach="instanceMatrix"
  count={matrices.length / 16}
  array={matrices}
  itemSize={16}
  usage={DynamicDrawUsage}
/>

<T.InstancedBufferAttribute
  attach="instanceColor"
  count={colors.length / 3}
  array={colors}
  itemSize={3}
  usage={DynamicDrawUsage}
/>

<slot />