# Transforms comparison

## Transforms comparison <a href="#transforms-comparison" id="transforms-comparison"></a>

Many of the transform operations available in the [`UnityEngine.Transform`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.html) class are available in the Entities package, with some key syntax differences.

### Unity engine transform property equivalents

| UnityEngine property                                                                                                    | ECS equivalent                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ----------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`childCount`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-childCount.html)                 | <p>Use <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Entities.SystemAPI.GetBuffer.html"><code>SystemAPI.GetBuffer</code></a>:<br><br></p><pre class="language-c#"><code class="lang-c#">
int childCount(ref SystemState state, Entity e)
{
  return SystemAPI.GetBuffer(e).Length;
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [`forward`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-forward.html)                       | <p>Use the Mathematics package <a href="https://docs.unity3d.com/Packages/com.unity.mathematics@1.2/api/Unity.Mathematics.math.normalize.html"><code>normalize</code></a> with <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalToWorld.Forward.html#Unity_Transforms_LocalToWorld_Forward"><code>LocalToWorld.Forward</code></a>. You can omit <code>normalize</code> if you know that the transform hierarchy doesn't have scale:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 forward(ref SystemState state, Entity e)
{
  return math.normalize(SystemAPI.GetComponent(e).Forward);
}

</code></pre>                                                                                                                                                                                                                                                                                  |
| [`localPosition`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-localPosition.html)           | <p>Use <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalTransform.Position.html"><code>LocalTransform.Position</code></a>:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 localPosition(ref SystemState state, Entity e)
{
  return SystemAPI.GetComponent(e).Position;
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| [`localRotation`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-localRotation.html)           | <p>Use <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalTransform.Rotation.html"><code>LocalTransform.Rotation</code></a>:<br><br></p><pre class="language-c#"><code class="lang-c#">
quaternion localRotation(ref SystemState state, Entity e)
{
  return SystemAPI.GetComponent(e).Rotation;
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| [`localScale`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-localScale.html)                 | <p>Use <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalTransform.Scale.html"><code>LocalTransform.Scale</code></a> and <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.PostTransformMatrix.html"><code>PostTransformMatrix</code></a>:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 localScale(ref SystemState state, Entity e)
{
  float scale = SystemAPI.GetComponent(e).Scale;
  if ( SystemAPI.HasComponent(e))
  {
    // If PostTransformMatrix contains skew, returned value will be inexact,
    // and diverge from GameObjects.
    float4x4 ptm = SystemAPI.GetComponent(e).Value;
    float lx = math.length(ptm.c0.xyz);
    float ly = math.length(ptm.c1.xyz);
    float lz = math.length(ptm.c2.xyz);
    return new float3(lx, ly, lz) * scale;
  }
  else
  {
    return new float3(scale, scale, scale);
  }
}

</code></pre> |
| [`localToWorldMatrix`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-localToWorldMatrix.html) | <p>Use <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalToWorld.Value.html"><code>LocalToWorld.Value</code></a>:<br><br></p><pre class="language-c#"><code class="lang-c#">
float4x4 localToWorldMatrix(ref SystemState state, Entity e)
{
  return SystemAPI.GetComponent(e).Value;
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| [`lossyScale`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-lossyScale.html)                 | <p>Use <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalToWorld.Value.html"><code>LocalToWorld.Value</code></a> and the Mathematics package <a href="https://docs.unity3d.com/Packages/com.unity.mathematics@1.2/api/Unity.Mathematics.math.length.html"><code>length</code></a>:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 lossyScale(ref SystemState state, Entity e)
{
  // If LocalToWorld contains skew, returned value will be inexact,
  // and diverge from GameObjects.
  float4x4 l2w = SystemAPI.GetComponent(e).Value;
  float lx = math.length(l2w.c0.xyz);
  float ly = math.length(l2w.c1.xyz);
  float lz = math.length(l2w.c2.xyz);
  return new float3(lx, ly, lz);
}

</code></pre>                                                                                                                                                                                 |
| [`parent`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-parent.html)                         | <p>Use <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.Parent.Value.html"><code>Parent.Value</code></a>:<br><br></p><pre class="language-c#"><code class="lang-c#">
Entity parent(ref SystemState state, Entity e)
{
  return SystemAPI.GetComponent(e).Value;
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| [`position`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-position.html)                     | <p>Use <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalToWorld.Position.html#Unity_Transforms_LocalToWorld_Position"><code>LocalToWorld.Position</code></a>:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 position(ref SystemState state, Entity e)
{
  return SystemAPI.GetComponent(e).Position;
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| [`right`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-right.html)                           | <p>Use the Mathematics package <a href="https://docs.unity3d.com/Packages/com.unity.mathematics@1.2/api/Unity.Mathematics.math.normalize.html"><code>normalize</code></a> with <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalToWorld.Right.html#Unity_Transforms_LocalToWorld_Right"><code>LocalToWorld.Right</code></a>. You can omit <code>normalize</code> if you know that the transform hierarchy doesn't have scale:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 right(ref SystemState state, Entity e)
{
  return math.normalize(SystemAPI.GetComponent(e).Right);
}

</code></pre>                                                                                                                                                                                                                                                                                            |
| [`root`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-root.html)                             | <p>Use <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.Parent.Value.html"><code>Parent.Value</code></a>:<br><br></p><pre class="language-c#"><code class="lang-c#">
Entity root(ref SystemState state, Entity e)
{
  while (SystemAPI.HasComponent(e))
  {
    e = SystemAPI.GetComponent(e).Value;
  }
  return e;
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| [`rotation`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-rotation.html)                     | <p>Use <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalToWorld.Value.html"><code>LocalToWorld.Value</code></a>:<br><br></p><pre class="language-c#"><code class="lang-c#">
quaternion rotation(ref SystemState state, Entity e)
{
  return SystemAPI.GetComponent(e).Value.Rotation();
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| [`up`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-up.html)                                 | <p>Use the Mathematics package <a href="https://docs.unity3d.com/Packages/com.unity.mathematics@1.2/api/Unity.Mathematics.math.normalize.html"><code>normalize</code></a> with <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalToWorld.Up.html#Unity_Transforms_LocalToWorld_Up"><code>LocalToWorld.Up</code></a>. You can omit <code>normalize</code> if you know that the transform hierarchy doesn't have scale:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 up(ref SystemState state, Entity e)
{
  return math.normalize(SystemAPI.GetComponent(e).Up);
}

</code></pre>                                                                                                                                                                                                                                                                                                           |
| [`worldToLocalMatrix`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-worldToLocalMatrix.html) | <p>Use the Mathematics package <a href="https://docs.unity3d.com/Packages/com.unity.mathematics@1.2/api/Unity.Mathematics.math.inverse.html"><code>inverse</code></a> with <a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/api/Unity.Transforms.LocalToWorld.Value.html"><code>LocalToWorld.Value</code></a>. You can omit <code>normalize</code> if you know that the transform hierarchy doesn't have scale:<br><br></p><pre class="language-c#"><code class="lang-c#">
float4x4 worldToLocalMatrix(ref SystemState state, Entity e)
{
  return math.inverse(SystemAPI.GetComponent(e).Value);
}

</code></pre>                                                                                                                                                                                                                                                                                                                       |

#### Properties with no equivalent

The following properties have no equivalent in the Entities package:

* [`eulerAngles`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-eulerAngles.html)
* [`localEulerAngles`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-localEulerAngles.html)
* [`hasChanged`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-hasChanged.html)
* [`hierarchyCapacity`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-hierarchyCapacity.html). Not needed. There is no limit to the number of children an entity can have.
* [`hierarchyCount`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform-hierarchyCount.html)

### Unity engine transform method equivalents

| UnityEngine method                                                                                                                        | ECS equivalent                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ----------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`DetachChildren`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.DetachChildren.html)                           | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
void DetachChildren(ref SystemState state, Entity e)
{
  DynamicBuffer buffer = SystemAPI.GetBuffer(e);
  state.EntityManager.RemoveComponent(buffer.AsNativeArray().Reinterpret(),
                                      ComponentType.ReadWrite());
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| [`GetChild`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.GetChild.html)                                       | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
Child child(ref SystemState state, Entity e, int index)
{
  return SystemAPI.GetBuffer(e)[index];
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| [`GetLocalPositionAndRotation`](https://docs.unity3d.com/ScriptReference/Transform.GetLocalPositionAndRotation.html)                      | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
void GetLocalPositionAndRotation(ref SystemState state, Entity e, out float3 localPosition, out quaternion localRotation)
{
  LocalTransform transform = SystemAPI.GetComponent(e);
  localPosition = transform.Position;
  localRotation = transform.Rotation;
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| [`GetPositionAndRotation`](https://docs.unity3d.com/ScriptReference/Transform.GetPositionAndRotation.html)                                | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
void GetPositionAndRotation(ref SystemState state, Entity e, out float3 position, out quaternion rotation)
{
  LocalToWorld l2w = SystemAPI.GetComponent(e);
  position = l2w.Value.Translation();
  rotation = l2w.Value.Rotation();
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| [`InverseTransformDirection`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.InverseTransformDirection.html)     | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 InverseTransformDirection(ref SystemState state, Entity e, float3 direction)
{
  LocalToWorld l2w = SystemAPI.GetComponent(e);
  return math.inverse(l2w.Value).TransformDirection(direction);
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| [`InverseTransformPoint`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.InverseTransformPoint.html)             | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 InverseTransformPoint(ref SystemState state, Entity e, float3 position)
{
  LocalToWorld l2w = SystemAPI.GetComponent(e);
  return math.inverse(l2w.Value).TransformPoint(worldPoint);
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| [`InverseTransformVector`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.InverseTransformVector.html)           | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 InverseTransformVector(ref SystemState state, Entity e, float3 vector)
{
  return math.inverse(SystemAPI.GetComponent(e).Value)
         .TransformDirection(vector);
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| [`IsChildOf`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.IsChildOf.html)                                     | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
bool IsChildOf(ref SystemState state, Entity e, Entity parent)
{
  return SystemAPI.HasComponent(e)
         &#x26;&#x26; SystemAPI.GetComponent(e).Value == parent;
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| [`LookAt`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.LookAt.html)                                           | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
void LookAt(ref SystemState state, Entity e, float3 target, float3 worldUp)
{
  if (SystemAPI.HasComponent(e))
  {
    Entity parent = SystemAPI.GetComponent(e).Value;
    float4x4 parentL2W = SystemAPI.GetComponent(parent).Value;
    target = math.inverse(parentL2W).TransformPoint(target);
  }
  LocalTransform transform = SystemAPI.GetComponent(e);
  quaternion rotation = quaternion.LookRotationSafe(target, worldUp);
  SystemAPI.SetComponent(e, transform.WithRotation(rotation));
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| [`Rotate`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.Rotate.html)                                           | <p>In the Entities transform system, rotation is always expressed as a quaternion, and an angle is always in radians. There is functionality in the Mathematics package library to convert Euler angles into quaternions, and to convert degrees into radians.</p><p>With <code>Space.Self</code>, or if the entity has no parent:<br><br></p><pre class="language-c#"><code class="lang-c#">
public void Rotate(ref SystemState state, Entity e, quaternion rotation)
{
  LocalTransform transform = SystemAPI.GetComponent(e);
  rotation = math.mul(rotation, transform.Rotation);
  SystemAPI.SetComponent(e, transform.WithRotation(rotation));
}

</code></pre><p>With <code>Space.World</code>, and the entity may have a parent:<br><br></p><pre class="language-c#"><code class="lang-c#">
void Rotate(ref SystemState state, Entity e, quaternion rotation)
{
  if (SystemAPI.HasComponent(e))
  {
    Entity parent = SystemAPI.GetComponent(e).Value;
    float4x4 parentL2W = SystemAPI.GetComponent(parent).Value;
    rotation = math.inverse(parentL2W).TransformRotation(rotation);
  }
  LocalTransform transform = SystemAPI.GetComponent(e);
  rotation = math.mul(rotation, transform.Rotation);
  SystemAPI.SetComponent(e, transform.WithRotation(rotation));
}

</code></pre> |
| [`RotateAround`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.RotateAround.html)                               | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
public void RotateAround(ref SystemState state, Entity e, float3 point, float3 axis, float angle)
{
  // Note: axis should be of unit length
  if (SystemAPI.HasComponent(e))
  {
    Entity parent = SystemAPI.GetComponent(e).Value;
    float4x4 parentL2W = SystemAPI.GetComponent(parent).Value;
    float4x4 invParentL2W = math.inverse(parentL2W);
    point = invParentL2W.TransformPoint(point);
    axis = invParentL2W.TransformDirection(axis);
  }
  var transform = SystemAPI.GetComponent(e);
  var q = quaternion.AxisAngle(axis, angle);
  transform.Position = point + math.mul(q, transform.Position - point);
  transform.Rotation = math.mul(q, transform.Rotation);
  SystemAPI.SetComponent(e, transform);
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| [`SetLocalPositionAndRotation`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.SetLocalPositionAndRotation.html) | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
void SetLocalPositionAndRotation(ref SystemState state, Entity e, float3 localPosition, quaternion localRotation)
{
  SystemAPI.SetComponent(e, LocalTransform.FromPositionRotation(localPosition, localRotation));
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| [`SetParent`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.SetParent.html)                                     | <p>Without <code>worldPositionStays</code>:<br><br></p><pre class="language-c#"><code class="lang-c#">
void SetParent(ref SystemState state, Entity e, Entity parent)
{
  SystemAPI.SetComponent(e, new Parent { Value = parent});
}

</code></pre><p>With <code>worldPositionStays</code>:<br><br></p><pre class="language-c#"><code class="lang-c#">
void SetParent(ref SystemState state, Entity e, Entity parent)
{
  float4x4 childL2W = SystemAPI.GetComponent(e).Value;
  float4x4 parentL2W = SystemAPI.GetComponent(parent).Value;
  float4x4 temp = math.mul(math.inverse(parentL2W), childL2W);

  SystemAPI.SetComponent(e, new Parent { Value = parent});
  SystemAPI.SetComponent(e, LocalTransform.FromMatrix(temp));
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| [`SetPositionAndRotation`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.SetPositionAndRotation.html)           | <p>If the entity has no parent:<br><br></p><pre class="language-c#"><code class="lang-c#">
void SetPositionAndRotationA(ref SystemState state, Entity e, float3 position, quaternion rotation)
{
  SystemAPI.SetComponent(e, LocalTransform.FromPositionRotation(position, rotation));
}

</code></pre><p>If the entity has a parent:<br><br></p><pre class="language-c#"><code class="lang-c#">
void SetPositionAndRotationB(ref SystemState state, Entity e, float3 position, quaternion rotation)
{
  if (SystemAPI.HasComponent(e))
  {
    Entity parent = SystemAPI.GetComponent(e).Value;
    float4x4 parentL2W = SystemAPI.GetComponent(parent).Value;
    float4x4 invParentL2W = math.inverse(parentL2W);
    position = invParentL2W.TransformPoint(position);
    rotation = invParentL2W.TransformRotation(rotation);
  }
  SystemAPI.SetComponent(e, LocalTransform.FromPositionRotation(position, rotation));
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                        |
| [`TransformDirection`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.TransformDirection.html)                   | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 TransformDirection(ref SystemState state, Entity e, float3 direction)
{
  float3 temp = SystemAPI.GetComponent(e).Value.TransformDirection(direction);
  return temp * (math.length(direction) / math.length(temp));
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [`TransformPoint`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.TransformPoint.html)                           | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 TransformPoint(ref SystemState state, Entity e, float3 position)
{
  return SystemAPI.GetComponent(e).Value.TransformPoint(position);
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| [`Transformvector`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.TransformVector.html)                         | <p>Use the following:<br><br></p><pre class="language-c#"><code class="lang-c#">
float3 TransformVector(ref SystemState state, Entity e, float3 vector)
{
  return SystemAPI.GetComponent(e).Value.TransformDirection(vector);
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| [`Translate`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.Translate.html)                                     | <p>With <code>Space.Self</code>, or if the entity has no parent:<br><br></p><pre class="language-c#"><code class="lang-c#">
void Translate(ref SystemState state, Entity e, float3 translation)
{
  SystemAPI.GetComponentRW(e, false).ValueRW.Position += translation;
}

</code></pre><p>With <code>Space.World</code>, and the entity may have a parent:<br><br></p><pre class="language-c#"><code class="lang-c#">
void Translate(ref SystemState state, Entity e, float3 translation)
{
  if (SystemAPI.HasComponent(e))
  {
    Entity parent = SystemAPI.GetComponent(e).Value;
    float4x4 parentL2W = SystemAPI.GetComponent(parent).Value;
    translation = math.inverse(parentL2W).TransformDirection(translation);
  }
  SystemAPI.GetComponentRW(e, false).ValueRW.Position += translation;
}

</code></pre>                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |

#### Methods with no equivalent

The following methods have no equivalent in the Entities package:

* [`Find`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.Find.html)
* [`GetSiblingIndex`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.GetSiblingIndex.html). Children are in arbitrary order.
* [`SetAsFirstSibling`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.SetAsFirstSibling.html). Children are in arbitrary order.
* [`SetAsLastSibling`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.SetAsLastSibling.html). Children are in arbitrary order.
* [`SetSiblingIndex`](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Transform.SetSiblingIndex.html). Children are in arbitrary order.
