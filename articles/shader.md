
跟随原文复习了一下game101辐射度量学的相关内容，还有对BRDF做了一些补充学习

![](https://pic1.zhimg.com/80/v2-9e3e3e4b71164a24b95cd44d9645cf0f_720w.jpg?source=ccfced1a)

添加图片注释，不超过 140 字（可选）

然后我想复刻一下原文中的shader。原文中的shader是GLSL写的，我把他写成了CG语言。

Shader "Custom/BRDF"
{
    Properties
    {
        _Albedo ("Color", Color) = (1,1,1,1)
        _MainTex ("Albedo (RGB)", 2D) = "white" {}
        _Roughness("Roughness", Range(0, 1)) = 0.0
        _Metallic("Metallic", Range(0, 1)) = 0.0
        _Ao("AO", float) = 0.0
    }
    SubShader
    {
        pass{
            Tags { "LightMode"="ForwardBase" }
            
                CGPROGRAM
                
                #pragma vertex vert
                #pragma fragment frag
                #pragma multi_compile_fwdbase
                #define PI 3.14159265359
                
                #include "Lighting.cginc"
                #include "AutoLight.cginc"
                
                fixed4 _Albedo;
                sampler2D _MainTex;
                float4 _MainTex_ST;

                float _Roughness;
                float _Metallic;
                float _Ao;

                fixed4 _Color;
                
                struct a2v {
                    float4 vertex : POSITION;
                    float3 normal : NORMAL;
                    float4 texcoord : TEXCOORD0;
                };
                
                struct v2f {
                    float3 worldNormal : TEXCOORD3;
                    float4 pos : SV_POSITION;
                    float4 worldPos :TEXCOORD4;
                    float4 uv : TEXCOORD0;
                    float3 viewDir : TEXCOORD2;
                };

                float DistributionGGX(float3 N, float3 H, float roughness)
                {
                    float a = roughness * roughness;
                    float a2 = a * a;
                    float NdotH = max(dot(N, H), 0.0);
                    float NdotH2 = NdotH * NdotH;

                    float nom = a2;
                    float denom = (NdotH2 * (a2 - 1.0) + 1.0);
                    denom = PI * denom * denom;

                    return nom / denom;
                }

                float GeometrySchlickGGX(float NdotV, float roughness)
                {
                    float r = (roughness + 1.0);
                    float k = (r*r) / 8.0;

                    float nom   = NdotV;
                    float denom = NdotV * (1.0 - k) + k;

                    return nom / denom;
                }

                float GeometrySmith(float3 N, float3 V, float3 L, float roughness)
                {
                    float NdotV = max(dot(N, V), 0.0);
                    float NdotL = max(dot(N, L), 0.0);
                    float ggx2 = GeometrySchlickGGX(NdotV, roughness);
                    float ggx1 = GeometrySchlickGGX(NdotL, roughness);

                    return ggx1 * ggx2;
                }

                float3 fresnelSchlick(float cosTheta, float3 F0)
                {
                    return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
                }

                v2f vert(a2v v) {
                    v2f o;
                    o.pos = UnityObjectToClipPos(v.vertex);
                    
                    o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;

                    
                    o.viewDir = WorldSpaceViewDir(v.vertex);
                    o.worldNormal = UnityObjectToWorldNormal(v.normal);
                    o.worldPos = mul(unity_ObjectToWorld, v.vertex );

                    return o;
                }
                
                fixed4 frag(v2f i) : SV_Target {	

                    fixed3 Lo = fixed3(0, 0, 0);
                    
                    //fixed3 lightDir = normalize(LightPos - i.worldPos);
                    fixed3 N = normalize(i.worldNormal);
                    fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Albedo.rgb;
                    float3 F0 = float3(0.04,0.04,0.04);
                    F0 = lerp(F0, albedo, _Metallic);	

                    // calculate per-light radiance	
                    float3 lightDir = float3(0, 0, 0) - i.worldPos;	
                    float distance = length(lightDir);
                    float attenuation = 1.0 / (distance * distance);

                   
                    float3 radiance = unity_LightColor[0] * attenuation;

                    // Cook-Torrance BRDF
                    
                    fixed3 LightDir = normalize(lightDir);
                    fixed3 ViewDir = normalize(i.viewDir);
                    fixed3 H = normalize(LightDir + ViewDir);
                    float NDF = DistributionGGX(N, H, _Roughness);
                    float G = GeometrySmith(N, ViewDir, LightDir, _Roughness);
                    float3 F = fresnelSchlick(clamp(dot(H, ViewDir), 0.0, 1.0), F0);

                    float3 nominator = NDF * G * F;
                    float denominator = 4 * max(dot(N, ViewDir), 0.0) * max(dot(N, LightDir), 0.0);
                    float3 specular = nominator / max(denominator, 0.001);

                    float3 kS = F;
                    float3 kD = float3(1, 1, 1) - kS;

                    kD *= 1.0 - _Metallic;

                    float NdotL = max(dot(N, LightDir), 0.0);

                    Lo += (kD * albedo / PI + specular) * radiance * NdotL;







                     lightDir = float3(9, 0, 0) - i.worldPos;	
                     distance = length(lightDir);
                    attenuation = 1.0 / (distance * distance);

                   
                     radiance = unity_LightColor[0] * attenuation;

                    // Cook-Torrance BRDF
                    
                    LightDir = normalize(lightDir);
                    ViewDir = normalize(i.viewDir);
                     H = normalize(LightDir + ViewDir);
                     NDF = DistributionGGX(N, H, _Roughness);
                     G = GeometrySmith(N, ViewDir, LightDir, _Roughness);
                     F = fresnelSchlick(clamp(dot(H, ViewDir), 0.0, 1.0), F0);

                     nominator = NDF * G * F;
                     denominator = 4 * max(dot(N, ViewDir), 0.0) * max(dot(N, LightDir), 0.0);
                     specular = nominator / max(denominator, 0.001);

                     kS = F;
                     kD = float3(1, 1, 1) - kS;

                    kD *= 1.0 - _Metallic;

                     NdotL = max(dot(N, LightDir), 0.0);

                    Lo += (kD * albedo / PI + specular) * radiance * NdotL;





                    lightDir = float3(0, 9, 0) - i.worldPos;	
                     distance = length(lightDir);
                    attenuation = 1.0 / (distance * distance);

                   
                     radiance = unity_LightColor[0] * attenuation;

                    // Cook-Torrance BRDF
                    
                    LightDir = normalize(lightDir);
                    ViewDir = normalize(i.viewDir);
                     H = normalize(LightDir + ViewDir);
                     NDF = DistributionGGX(N, H, _Roughness);
                     G = GeometrySmith(N, ViewDir, LightDir, _Roughness);
                     F = fresnelSchlick(clamp(dot(H, ViewDir), 0.0, 1.0), F0);

                     nominator = NDF * G * F;
                     denominator = 4 * max(dot(N, ViewDir), 0.0) * max(dot(N, LightDir), 0.0);
                     specular = nominator / max(denominator, 0.001);

                     kS = F;
                     kD = float3(1, 1, 1) - kS;

                    kD *= 1.0 - _Metallic;

                     NdotL = max(dot(N, LightDir), 0.0);

                    Lo += (kD * albedo / PI + specular) * radiance * NdotL;




                    lightDir = float3(9, 9, 0) - i.worldPos;	
                     distance = length(lightDir);
                    attenuation = 1.0 / (distance * distance);

                   
                     radiance = unity_LightColor[0] * attenuation;

                    // Cook-Torrance BRDF
                    
                    LightDir = normalize(lightDir);
                    ViewDir = normalize(i.viewDir);
                     H = normalize(LightDir + ViewDir);
                     NDF = DistributionGGX(N, H, _Roughness);
                     G = GeometrySmith(N, ViewDir, LightDir, _Roughness);
                     F = fresnelSchlick(clamp(dot(H, ViewDir), 0.0, 1.0), F0);

                     nominator = NDF * G * F;
                     denominator = 4 * max(dot(N, ViewDir), 0.0) * max(dot(N, LightDir), 0.0);
                     specular = nominator / max(denominator, 0.001);

                     kS = F;
                     kD = float3(1, 1, 1) - kS;

                    kD *= 1.0 - _Metallic;

                     NdotL = max(dot(N, LightDir), 0.0);

                    Lo += (kD * albedo / PI + specular) * radiance * NdotL;




                

                    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo * 0.003;
                    fixed3 color = ambient + Lo;

                    color = color / (color + fixed3(1, 1, 1));

                    color = pow(color, fixed3(1.0 / 2.2, 1.0 / 2.2, 1.0 / 2.2));


                    return fixed4(color + ambient, 1.0);
                }
                
                ENDCG
            }
            //FallBack "Diffuse"
    }
    
}


是这个代码的原理并不难理解，主要是对辐射度量学的理解，还有求积分利用黎曼积分转化成近似值的形式。

我在写代码过程中遇到的主要问题是unity对光源支持比较麻烦，ForwaBase只支持一盏主要的光源，也有可能是我不知道怎么使用其他的光源。Vertex虽然支持4盏，但是效果比较差。我最终是在代码中自定义光源位置和数量来实现多点光源。最终效果如下图：

![](https://pic1.zhimg.com/80/v2-ed3ae597d5e78003b0524c3e0f6f746f_720w.png?source=ccfced1a)

上到下金属性减弱，左到右粗糙度减小
