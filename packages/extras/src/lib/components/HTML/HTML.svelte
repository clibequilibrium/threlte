<script lang="ts">
  import {
    createRawEventDispatcher,
    forwardEventHandlers,
    T,
    useTask,
    useThrelte
  } from '@threlte/core'
  import {
    Vector3,
    Group,
    Object3D,
    OrthographicCamera,
    Raycaster,
    DoubleSide,
    Mesh,
    PerspectiveCamera,
    Matrix4
  } from 'three'
  import { useHasEventListeners } from '../../hooks/useHasEventListeners'
  import {
    defaultCalculatePosition,
    epsilon,
    getCameraCSSMatrix,
    getViewportFactor,
    getObjectCSSMatrix,
    isObjectBehindCamera,
    isObjectVisible,
    objectScale,
    objectZIndex
  } from './utils'
  import type { HTMLEvents, HTMLProps, HTMLSlots } from './HTML.svelte'
  import VertexShader from './vertex'

  type $$Props = HTMLProps
  type $$PropsWithDefaults = Required<$$Props>
  type $$Events = HTMLEvents
  type $$Slots = HTMLSlots

  // Group Properties
  export let transform: $$PropsWithDefaults['transform'] = false
  export let calculatePosition: $$PropsWithDefaults['calculatePosition'] = defaultCalculatePosition
  export let eps: $$PropsWithDefaults['eps'] = 0.001
  export let occlude: $$PropsWithDefaults['occlude'] = false
  export let zIndexRange: $$PropsWithDefaults['zIndexRange'] = [16777271, 0]
  export let sprite: $$PropsWithDefaults['sprite'] = false
  export let pointerEvents: $$PropsWithDefaults['pointerEvents'] = 'auto'
  export let center: $$PropsWithDefaults['center'] = false
  export let fullscreen: $$PropsWithDefaults['fullscreen'] = false
  export let distanceFactor: $$Props['distanceFactor'] | undefined = undefined
  export let as: $$PropsWithDefaults['as'] = 'div'
  export let prepend: $$PropsWithDefaults['prepend']
  export let onOcclude: $$PropsWithDefaults['onOcclude']
  export let castShadow: $$PropsWithDefaults['castShadow']
  export let receiveShadow: $$PropsWithDefaults['receiveShadow']
  export let material: $$PropsWithDefaults['material']
  export let geometry: $$PropsWithDefaults['geometry']
  export let wrapperClass: $$PropsWithDefaults['wrapperClass']
  export let portal: $$Props['portal'] | undefined = undefined

  const { renderer, camera, scene, size } = useThrelte()

  export let ref = new Group()
  const group = new Group()

  let element = document.createElement(as)
  let oldZoom = 0
  let oldPosition = [0, 0]
  let transformOuterRef: HTMLDivElement
  let transformInnerRef: HTMLDivElement
  let isMeshSizeSet = false
  let visible = false

  const occlusionMesh = new Mesh()
  const raycaster = new Raycaster()

  $: pos = calculatePosition(ref, camera.current, $size)
  $: vertexShader = transform ? undefined : VertexShader

  $: isRayCastOcclusion =
    (occlude && occlude !== 'blending') || (Array.isArray(occlude) && occlude.length > 0)

  $: matrix = new Matrix4()
  $: width = $size.width
  $: height = $size.height
  $: halfWidth = width / 2
  $: halfHeight = height / 2
  $: fov = $camera.projectionMatrix.elements[5] * halfHeight
  $: viewportFactor = getViewportFactor($camera, new Vector3(), $size)

  $: {
    if (wrapperClass) element.className = wrapperClass
  }

  let oldZIndex = ''

  $: {
    const canvas = renderer.domElement

    if (occlude && occlude === 'blending') {
      oldZIndex = canvas.style.zIndex
      canvas.style.zIndex = `${Math.floor(zIndexRange[0] / 2)}`
      canvas.style.position = 'absolute'
      canvas.style.pointerEvents = 'none'
    } else {
      canvas.style.zIndex = oldZIndex
      canvas.style.removeProperty('position')
      canvas.style.pointerEvents = null!
    }
  }

  useTask(() => {
    camera.current.updateMatrixWorld()
    group.updateWorldMatrix(true, false)
    const vec = transform ? oldPosition : calculatePosition(group, camera.current, $size)

    if (
      transform ||
      Math.abs(oldZoom - camera.current.zoom) > eps ||
      Math.abs(oldPosition[0] - vec[0]) > eps ||
      Math.abs(oldPosition[1] - vec[1]) > eps
    ) {
      const isBehindCamera = isObjectBehindCamera(group, camera.current)
      let raytraceTarget: null | undefined | boolean | Object3D[] = false

      if (isRayCastOcclusion) {
        if (Array.isArray(occlude)) {
          raytraceTarget = occlude as Object3D[]
        } else if (occlude !== 'blending') {
          raytraceTarget = [scene]
        }
      }

      const previouslyVisible = visible

      if (raytraceTarget) {
        const isvisible = isObjectVisible(group, camera.current, raycaster, raytraceTarget)
        visible = isvisible && !isBehindCamera
      } else {
        visible = !isBehindCamera
      }

      if (previouslyVisible !== visible) {
        if ($$restProps.visibilitychange) {
          $$restProps.visibilitychange(visible)
        } else {
          element.style.display = visible ? 'block' : 'none'
        }
      }

      const halfRange = Math.floor(zIndexRange[0] / 2)
      const zRange = occlude
        ? isRayCastOcclusion //
          ? [zIndexRange[0], halfRange]
          : [halfRange - 1, 0]
        : zIndexRange

      element.style.zIndex = `${objectZIndex(
        group,
        camera.current as OrthographicCamera | PerspectiveCamera,
        zRange
      )}`

      if (transform) {
        const { isOrthographicCamera, top, left, bottom, right } =
          camera.current as OrthographicCamera
        const cameraMatrix = getCameraCSSMatrix(camera.current.matrixWorldInverse)
        const cameraTransform = isOrthographicCamera
          ? `scale(${fov})translate(${epsilon(-(right + left) / 2)}px,${epsilon(
              (top + bottom) / 2
            )}px)`
          : `translateZ(${fov}px)`

        if (sprite) {
          matrix
            .copy(camera.current.matrixWorldInverse)
            .transpose()
            .copyPosition(matrix)
            .scale(group.scale)
          matrix.elements[3] = matrix.elements[7] = matrix.elements[11] = 0
          matrix.elements[15] = 1
        } else {
          matrix.copy(group.matrixWorld)
        }

        element.style.width = `${width}px`
        element.style.height = `${height}px`
        element.style.perspective = isOrthographicCamera ? '' : `${fov}px`
        transformOuterRef.style.transform = `${cameraTransform}${cameraMatrix}translate(${halfWidth}px,${halfHeight}px)`
        transformInnerRef.style.transform = getObjectCSSMatrix(
          matrix,
          1 / ((distanceFactor || 10) / 400)
        )
      } else {
        const scale =
          distanceFactor === undefined ? 1 : objectScale(group, camera.current) * distanceFactor
        element.style.transform = `translate3d(${vec[0]}px,${vec[1]}px,0) scale(${scale})`
      }
      oldPosition = vec
      oldZoom = camera.current.zoom
    }

    if (!isRayCastOcclusion && !isMeshSizeSet) {
      if (transform) {
        const el = transformOuterRef.children[0]

        // console.log(el.clientWidth, el.clientHeight);
        if (el?.clientWidth && el?.clientHeight) {
          const { isOrthographicCamera } = camera.current as OrthographicCamera

          if (isOrthographicCamera || geometry) {
            const { scale } = $$restProps
            if (scale) {
              if (!Array.isArray(scale)) {
                occlusionMesh.scale.setScalar(1 / scale)
              } else {
                occlusionMesh.scale.set(1 / scale[0], 1 / scale[1], 1 / scale[2])
              }
            }
          } else {
            const ratio = (distanceFactor ?? 10) / 400
            const w = el.clientWidth * ratio
            const h = el.clientHeight * ratio

            occlusionMesh.scale.set(w, h, 1)
          }

          isMeshSizeSet = true
        }
      } else {
        const el = element.children[0]

        if (el?.clientWidth && el?.clientHeight) {
          const ratio = 1 / viewportFactor
          const w = el.clientWidth * ratio
          const h = el.clientHeight * ratio

          occlusionMesh.scale.set(w, h, 1)

          isMeshSizeSet = true
        }

        occlusionMesh.lookAt(camera.current.position)
      }
    }
  })

  const portalAction = (el: HTMLElement) => {
    const target = portal ?? renderer.domElement.parentElement?.parentElement
    if (!target) {
      console.warn('<HTML>: target is undefined.')
      return
    }
    target.append(el)
    return {
      destroy: () => el.remove()
    }
  }

  const component = forwardEventHandlers()

  const logVertex = `
#include <common>
#include <logdepthbuf_pars_vertex>
void main() {
    gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
    #include <logdepthbuf_vertex>
}
`

  const logFragment = `
#include <logdepthbuf_pars_fragment>

void main() {

	#include <logdepthbuf_fragment>
	gl_FragColor = vec4(0.0,0.0,0.0,0.0);
	
}
`
</script>

<T
  is={group}
  bind:ref
  {...$$restProps}
  bind:this={$component}
>
  {#if occlude && !isRayCastOcclusion}
    <T
      is={occlusionMesh}
      {castShadow}
      {receiveShadow}
    >
      {#if geometry}
        <T is={geometry} />
      {:else}
        <T.PlaneGeometry />
      {/if}

      {#if material}
        <T is={material} />
      {:else if !transform}
        <T.ShaderMaterial
          side={DoubleSide}
          {vertexShader}
          fragmentShader={`void main(){ gl_FragColor=vec4(0.0, 0.0, 0.0, 0.0); }`}
        />
      {:else}
        <T.ShaderMaterial
          side={DoubleSide}
          vertexShader={logVertex}
          fragmentShader={logFragment}
        />
      {/if}
    </T>
  {/if}
</T>

<svelte:element
  this={as}
  use:portalAction
  bind:this={element}
  style:position="absolute"
  style:top="0"
  style:left="0"
  style:pointer-events={transform ? 'none' : undefined}
  style:overflow={transform ? 'hidden' : undefined}
  style:transform={transform ? undefined : `translate3d(${pos[0]}px,${pos[1]}px,0)`}
  style:transform-origin={transform ? undefined : '0 0'}
>
  {#if transform}
    <div
      bind:this={transformOuterRef}
      style:position="absolute"
      style:top="0"
      style:left="0"
      style:transform-style="preserve-3d"
      style:pointer-events="none"
      style:width={`${width}px`}
      style:height={`${height}px`}
    >
      <div
        bind:this={transformInnerRef}
        style:position="absolute"
        style:pointer-events={pointerEvents}
      >
        <div
          class={$$restProps.class}
          style={$$restProps.style}
        >
          <slot />
        </div>
      </div>
    </div>
  {:else}
    <div
      style:position="absolute"
      style:transform={center ? 'translate3d(-50%,-50%,0)' : 'none'}
      style:top={fullscreen ? `${-height / 2}px` : undefined}
      style:left={fullscreen ? `${-width / 2}px` : undefined}
      style:width={fullscreen ? `${width / 2}px` : undefined}
      style:height={fullscreen ? `${height}px` : undefined}
      style={$$restProps.style}
      class={$$restProps.class}
    >
      <slot />
    </div>
  {/if}
</svelte:element>
