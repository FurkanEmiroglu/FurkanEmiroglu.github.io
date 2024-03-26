Showcase
--------------
![Showcase](https://raw.githubusercontent.com/FurkanEmiroglu/FurkanEmiroglu/main/Gaussian-Blur-PostProcess/gaussian-blur.gif)
--------------



```HLSL
Shader "PostProcessing/GaussianBlurFullScreen"
{
    Properties
    {
        [HideInInspector]
        _MainTex ("Texture", 2D) = "white" {}

        [HideInInspector]
        _Spread ("Spread", Float) = 0

        [HideInInspector]
        _GridSize ("Grid Size", Integer) = 0
    }
    SubShader
    {
        Tags
        {
            "RenderType"="Opaque"
            "RenderPipeline" = "UniversalPipeline"
        }

        HLSLINCLUDE
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

        #define E 2.71828f

        TEXTURE2D(_MainTex);
        SAMPLER(sampler_MainTex);

        CBUFFER_START(UnityPerMaterial)
            float4 _MainTex_TexelSize;
            uint _Grid;
            float _Spread;
        CBUFFER_END

        struct appdata
        {
            float4 positionOS : POSITION;
            float2 uv : TEXCOORD0;
        };

        struct interpolators
        {
            float2 uv : TEXCOORD0;
            float4 positionCS : SV_POSITION;
        };

        float gaussian(const int param)
        {
            const float sigma_sqr = _Spread * _Spread;
            return (1 / sqrt(TWO_PI * sigma_sqr) * pow(E, -(param * param) / (2 * sigma_sqr)));
        }

        interpolators vert(appdata v)
        {
            interpolators o;
            o.positionCS = TransformObjectToHClip(v.positionOS.xyz);
            o.uv = v.uv;
            return o;
        }

        float4 frag_horizontal(const interpolators input) : SV_Target
        {
            float3 col = 0;
            float grid_sum = 0.0f;

            // how many pixels are there left / right of this fragment
            const int upper = (_Grid - 1) / 2;
            const int lower = -upper;

            for (int x = lower; x <= upper; ++x)
            {
                const float g = gaussian(x);
                const float2 uv_offset_x = float2(input.uv.x + _MainTex_TexelSize.x * x, input.uv.y);
                grid_sum += g;
                col += g * SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv_offset_x).xyz;
            }
            col /= grid_sum;
            return float4(col, 1);
        }

        float4 frag_vertical(const interpolators input) : SV_Target
        {
            float3 col = 0;
            float grid_sum = 0.0f;

            // how many pixels are there up / down of this fragment
            const int upper = (_Grid - 1) / 2;
            const int lower = -upper;

            for (int y = lower; y <= upper; ++y)
            {
                const float g = gaussian(y);
                const float2 uv_offset_y = float2(input.uv.x, input.uv.y + _MainTex_TexelSize.y * y);
                grid_sum += g;
                col += g * SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv_offset_y).xyz;
            }
            col /= grid_sum;
            return float4(col, 1);
        }
        ENDHLSL

        Pass
        {
            Name "Horizontal"

            HLSLPROGRAM
            #pragma vertex vert;
            #pragma fragment frag_horizontal;
            ENDHLSL
        }

        Pass
        {
            Name "Vertical"

            HLSLPROGRAM
            #pragma vertex vert;
            #pragma fragment frag_vertical;
            ENDHLSL
        }
    }
}
```

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class BlurRenderPass : ScriptableRenderPass
{
    private Material _material;
    private BlurPostProcessSettings _blurPostProcessSettings;
        
    private RenderTargetIdentifier _source;
    private RenderTargetHandle _blurTexture;
    private int _blurTextureID;
    
    private static readonly int GridSize = Shader.PropertyToID("_Grid");
    private static readonly int Spread = Shader.PropertyToID("_Spread");

    public bool Setup(ScriptableRenderer renderer)
    {
        _source = renderer.cameraColorTarget;
        _blurPostProcessSettings = VolumeManager.instance.stack.GetComponent<BlurPostProcessSettings>();
        renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;

        if (_blurPostProcessSettings != null && _blurPostProcessSettings.IsActive())
        {
            _material = new Material(Shader.Find("PostProcessing/GaussianBlurFullScreen"));
            return true;
        }

        return false;
    }

    public override void Configure(CommandBuffer cmd, RenderTextureDescriptor cameraTexDesc)
    {
        if (_blurPostProcessSettings == null || !_blurPostProcessSettings.IsActive()) return;

        _blurTextureID = Shader.PropertyToID("_TemporaryBlurTextureID");
        _blurTexture = new RenderTargetHandle();
        _blurTexture.id = _blurTextureID;
        cmd.GetTemporaryRT(_blurTexture.id, cameraTexDesc);
            
        base.Configure(cmd, cameraTexDesc);
    } 
        
    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        if (_blurPostProcessSettings == null || !_blurPostProcessSettings.IsActive()) return;

        CommandBuffer cmd = CommandBufferPool.Get("Blur Post Process");
        int gridSize = Mathf.CeilToInt(_blurPostProcessSettings.GetBlurValue());
            
        if (gridSize % 2 == 0)
        {
            gridSize++;
        }
        _material.SetInteger(GridSize, gridSize);
        _material.SetFloat(Spread, _blurPostProcessSettings.GetBlurValue());
           
        cmd.Blit(_source, _blurTexture.id, _material, 0);
        cmd.Blit(_blurTexture.id, _source, _material, 1);
        
        context.ExecuteCommandBuffer(cmd);
        
        cmd.Clear();
        CommandBufferPool.Release(cmd);
    }

    public override void FrameCleanup(CommandBuffer cmd)
    {
        cmd.ReleaseTemporaryRT(_blurTextureID);
        base.FrameCleanup(cmd);
    }
}
```
```csharp
using UnityEngine.Rendering.Universal;

public class BlurRendererFeature : ScriptableRendererFeature
{
    private BlurRenderPass _renderPass;
    
    public override void Create()
    {
        _renderPass = new BlurRenderPass();
        name = "Blur";
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (_renderPass.Setup(renderer))
        {
            renderer.EnqueuePass(_renderPass);
        }
    }
}
```
```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

[System.Serializable, VolumeComponentMenu("Gaussian Blur")]
public class BlurPostProcessSettings : VolumeComponent, IPostProcessComponent
{
    [Tooltip("Standard Deviation (spread) of the blur. Grid size is approx. 3x larger.")]
    [SerializeField]
    private ClampedFloatParameter strength = new(0.0f, MinStrength, MaxStrength);

    private const float MinStrength = 0.001f;
    private const float MaxStrength = 15.0f;
    
    public bool IsActive()
    {
        return strength.value > MinStrength && active;
    }

    public bool IsTileCompatible()
    {
        return false;
    }

    public void SetBlurStrength(float str)
    {
        strength.SetValue(new FloatParameter(Mathf.Clamp(str, MinStrength, MaxStrength)));
        strength.SetValue(new FloatParameter(str));
    }
    
    public float GetBlurValue()
    {
        return strength.value * 6.0f;
    }
}
```




How To Install

--------------
This package uses the [scoped registry] feature to resolve package
dependencies. Open the Package Manager page in the Project Settings window and
add the following entry to the Scoped Registries list:

### 1. Open Edit / Project Settings in Unity
![Project Settings](https://raw.githubusercontent.com/FurkanEmiroglu/FurkanEmiroglu/main/Project%20Settings.jpg)

### 2. Go to Package Manager
![Package Manager](https://raw.githubusercontent.com/FurkanEmiroglu/FurkanEmiroglu/main/Package%20Manager.jpg)

### 3. On Scoped Registries Window add the keys below
![Scoped Registry](https://raw.githubusercontent.com/FurkanEmiroglu/FurkanEmiroglu/main/Scoped%20Registries.jpg)
- Name: Enter any name here
- URL: `https://registry.npmjs.com`
- Scope: `com.femiroglu`

### 4. Open Window / Package Manager in Unity

![Open Package Manager](https://raw.githubusercontent.com/FurkanEmiroglu/FurkanEmiroglu/main/Open%20Package%20Manager.jpg)

### 5. Select My Registries Option
![My Registries](https://raw.githubusercontent.com/FurkanEmiroglu/FurkanEmiroglu/main/Select%20My%20Registries.jpg)

Now you can install the package from My Registries page in the Package Manager
window.