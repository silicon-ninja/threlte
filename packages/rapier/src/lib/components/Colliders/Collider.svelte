<script lang="ts">
  import {
    ActiveCollisionTypes,
    ActiveEvents,
    CoefficientCombineRule,
    Collider,
    ColliderDesc,
    type RigidBody
  } from '@dimforge/rapier3d-compat'
  import {
    SceneGraphObject,
    TransformableObject,
    useFrame,
    type Position,
    type Rotation,
    type TransformableObjectProperties
  } from '@threlte/core'
  import { createEventDispatcher, onDestroy, onMount, tick } from 'svelte'
  import { Object3D, Quaternion, Vector3 } from 'three'
  import { useCollisionGroups } from '../../hooks/useCollisionGroups'
  import { useHasEventListeners } from '../../hooks/useHasEventListener'
  import { useRapier } from '../../hooks/useRapier'
  import { useRigidBody } from '../../hooks/useRigidBody'
  import { applyColliderActiveEvents } from '../../lib/applyColliderActiveEvents'
  import { applyTransforms } from '../../lib/applyTransforms'
  import { getWorldPosition, getWorldQuaternion } from '../../lib/getWorldTransforms'
  import { positionToVector3 } from '../../lib/positionToVector3'
  import { rotationToQuaternion } from '../../lib/rotationToQuaternion'
  import { scaleColliderArgs } from '../../lib/scaleColliderArgs'
  import type { ColliderEventMap } from '../../types/types'

  type Shape = $$Generic<
    | 'ball'
    | 'capsule'
    | 'segment'
    | 'triangle'
    | 'roundTriangle'
    | 'polyline'
    | 'trimesh'
    | 'cuboid'
    | 'roundCuboid'
    | 'heightfield'
    | 'cylinder'
    | 'roundCylinder'
    | 'cone'
    | 'roundCone'
    | 'convexHull'
    | 'convexMesh'
    | 'roundConvexHull'
    | 'roundConvexMesh'
  >

  type Args = $$Generic<Parameters<typeof ColliderDesc[Shape]>>

  interface Props {
    shape: Shape
    args: Args
    position?: NonNullable<TransformableObjectProperties['position']>
    rotation?: TransformableObjectProperties['rotation']
    scale?: NonNullable<TransformableObjectProperties['scale']>
    lookAt?: NonNullable<TransformableObjectProperties['lookAt']>
    restitution?: number
    restitutionCombineRule?: CoefficientCombineRule
    friction?: number
    frictionCombineRule?: CoefficientCombineRule
    sensor?: boolean
    collider?: Collider
    contactForceEventThreshold?: number
  }

  type Density = $$Generic<number | undefined>
  type Mass = $$Generic<Density extends undefined ? number | undefined : undefined>
  type MassProperties = $$Generic<
    Density extends undefined
      ? Mass extends undefined
        ? {
            mass: number
            centerOfMass: Position
            principalAngularInertia: Position
            angularInertiaLocalFrame: Rotation
          }
        : undefined
      : undefined
  >

  interface WithDensity extends Props {
    density?: Density
  }

  interface WithMass extends Props {
    mass?: Mass
  }

  interface WithMassProperties extends WithMass {
    massProperties?: MassProperties
  }

  type $$Props = WithDensity | WithMass | WithMassProperties

  export let shape: $$Props['shape']
  export let args: $$Props['args']
  export let position: $$Props['position'] = undefined
  export let rotation: $$Props['rotation'] = undefined
  export let scale: $$Props['scale'] = undefined
  export let lookAt: $$Props['lookAt'] = undefined
  export let restitution: $$Props['restitution'] = undefined
  export let restitutionCombineRule: $$Props['restitutionCombineRule'] = undefined
  export let friction: $$Props['friction'] = undefined
  export let frictionCombineRule: $$Props['frictionCombineRule'] = undefined
  export let sensor: $$Props['sensor'] = undefined
  export let contactForceEventThreshold: $$Props['contactForceEventThreshold'] = undefined

  export let density = undefined as Density
  export let mass = undefined as Mass
  export let massProperties = undefined as MassProperties

  const object = new Object3D()

  /**
   * Immediately apply transforms
   */
  applyTransforms(object, position, rotation, scale, lookAt)
  object.updateWorldMatrix(true, false)

  const rigidBody = useRigidBody()
  const isAttached = !!rigidBody

  const rapierContext = useRapier()
  const { world } = rapierContext

  export let collider: Collider | undefined = undefined

  const collisionGroups = useCollisionGroups()

  /**
   * Events setup
   */
  type $$Events = {
    [key in keyof ColliderEventMap]: CustomEvent<ColliderEventMap[key]>
  }
  const dispatcher = createEventDispatcher<ColliderEventMap>()

  /**
   * Actual collider setup happens onMount as only then
   * the transforms are finished.
   */
  onMount(async () => {
    await tick()
    const scale = object.getWorldScale(new Vector3())
    const scaledArgs = scaleColliderArgs(shape, args, scale)

    // @ts-ignore
    const colliderDesc = ColliderDesc[shape](...scaledArgs) as ColliderDesc

    collider = world.createCollider(colliderDesc, rigidBody)

    /**
     * Add collider to context
     */
    rapierContext.addColliderToContext(collider, object, dispatcher)

    /**
     * For use in conjunction with component <CollisionGroups>
     */
    collisionGroups.registerColliders([collider])

    if (isAttached) {
      const rigidBodyWorldPos = new Vector3()
      const rigidBodyWorldQuatInversed = new Quaternion()
      object.traverseAncestors((child: Object3D) => {
        if (child.userData.isRigidBody) {
          child.getWorldPosition(rigidBodyWorldPos)
          child.getWorldQuaternion(rigidBodyWorldQuatInversed)
          rigidBodyWorldQuatInversed.invert()
        }
      })
      const worldPosition = object.getWorldPosition(new Vector3()).sub(rigidBodyWorldPos)
      const worldRotation = object
        .getWorldQuaternion(new Quaternion())
        .premultiply(rigidBodyWorldQuatInversed)
      collider.setTranslationWrtParent(worldPosition)
      collider.setRotationWrtParent(worldRotation)
    } else {
      collider.setTranslation(object.getWorldPosition(new Vector3()))
      collider.setRotation(object.getWorldQuaternion(new Quaternion()))
    }
  })

  const { hasEventListeners: colliderHasEventListeners } = useHasEventListeners<typeof dispatcher>()

  $: {
    if (collider) {
      applyColliderActiveEvents(
        collider,
        colliderHasEventListeners,
        rigidBody?.userData?.hasEventListeners
      )
      collider.setActiveCollisionTypes(ActiveCollisionTypes.ALL)
      collider.setRestitution(restitution ?? 0)
      collider.setContactForceEventThreshold(1)
      collider.setRestitutionCombineRule(restitutionCombineRule ?? CoefficientCombineRule.Average)
      collider.setFriction(friction ?? 0.7)
      collider.setFrictionCombineRule(frictionCombineRule ?? CoefficientCombineRule.Average)
      collider.setSensor(sensor ?? false)
      collider.setContactForceEventThreshold(contactForceEventThreshold ?? 0)
      if (density) collider.setDensity(density)
      if (mass) collider.setMass(mass)
      if (massProperties)
        collider.setMassProperties(
          massProperties.mass,
          positionToVector3(massProperties.centerOfMass),
          positionToVector3(massProperties.principalAngularInertia),
          rotationToQuaternion(massProperties.angularInertiaLocalFrame)
        )
    }
  }

  useFrame(
    () => {
      if (!collider) return
      collider.setTranslation(getWorldPosition(object))
      collider.setRotation(getWorldQuaternion(object))
    },
    {
      autostart: !isAttached
    }
  )

  /**
   * Cleanup
   */
  onDestroy(() => {
    if (!collider) return
    rapierContext.removeColliderFromContext(collider)
    collisionGroups.removeColliders([collider])
    world.removeCollider(collider, true)
  })
</script>

<SceneGraphObject {object}>
  <slot />
</SceneGraphObject>

{#if !isAttached}
  <TransformableObject {object} {position} {rotation} {scale} />
{/if}
