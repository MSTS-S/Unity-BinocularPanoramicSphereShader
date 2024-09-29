# Unity-BinocularPanoramicSphereShader
This Unity3D shader supports "stereo vision" in panoramic spheres when using an HMD (Head-Mounted Display).
<br><br>



## 概要
BinocularPanoramicSphereShaderはUnity3Dで使用可能な両眼対応全天球シェーダーです．Sphereモデルを天球として利用する場合における全天球シェーダーは一部のwebサイトで公開されています．しかしそのようなシェーダーはUnity Editor上では正常に動作するものの，HMD (Head Mounted Display) から見たときに片目（左眼）のみ表示されるという問題が確認されました（使用したデバイス：[Meta Quest2](https://www.meta.com/jp/quest/products/quest-2/)）．そこでUnity3Dにおいて，HMDを着用して視聴した場合に，両目に正常に表示されるSharderを作成しました．
<br><br>

## 全天球シェーダー（Sphereモデル）
### PanoramicSphereShader.sharder

Sphereモデルを天球として利用する場合における全天球シェーダーは以下の手順により作成されます．

1. Projectビューで右クリック > Create > Sharder > Unlit Sharder
2. 名前を変更（今回の場合はPanoramicSphereShader）
3. SubSharderに次の一文を追記
    - Cull Front
4. v2f vert(appdata v)の1行目に次の一文を追記
    - v.uv.x = 1 - v.uv.x;

上記1～4の手順を踏んだシェーダー全体のスクリプトは次のようになります．なお，追記箇所にはコメントアウトで**added**と記してあります．

```
Shader "Unlit/PanoramicSphereShader"
{
    Properties
    {
        _MainTex("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType" = "Opaque" }
        Cull Front // added
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                UNITY_FOG_COORDS(1)
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;

            v2f vert(appdata v)
            {
                v.uv.x = 1 - v.uv.x; // added
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                UNITY_TRANSFER_FOG(o,o.vertex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                // sample the texture
                fixed4 col = tex2D(_MainTex, i.uv);
                // apply fog
                UNITY_APPLY_FOG(i.fogCoord, col);
                return col;
            }
            ENDCG
        }
    }
}
```
<br><br>


## 片目（左目）でのみ描画される問題
[Unity Documentation](https://docs.unity3d.com/ja/2022.3/Manual/SinglePassStereoRendering.html)によると，Unity XRでは次の2つのStereo Rendering Modeが提供されています．

- Multi-pass
    - 左右の目それぞれに対して1つのレンダリングパスが実行．
    - レンダリングループの一部が2つのパス間で共有されるため、2つのカメラでSceneをレンダリングするよりも高速．
    - 既存のシェーダーやレンダリングユーティリティとの互換性が最も高い．
    - Single-pass Instanced Mode よりも低速．

- Single-pass Instanced
    - インスタンス化されたドローコールを使用して、シングルパスでシーンをレンダリング．
    - Multi-pass Modeと比較し，CPU使用量の大幅削減に加え，GPU使用量が若干削減．
    - Multiviewは一部のOpenGLおよびOpenGL ESデバイスでサポートされており，このMultiviewが利用可能である場合にのみSingle-pass Instancedがこれに置き換えられる．

実際にUnityでProjectを作成した後に，Edit > Project Settings > XR Plug-in Management > Oculus を選択後，Stereo Rendering Modeを確認するとdefaultでMultiview(Single-pass Instanced)に設定されていることが確認できます(XR Plug-in Management及びOculusはインストール必須です)．

![Project Settingのスクリーンショット](Document/ProjectSetting.png)

ですが，このMultiview(Single-pass Instanced)には大きな落とし穴があります．

> [!Note]
> One common issue with Single Pass Instanced Rendering occurs if developers already have existing custom shaders not written for instancing. After enabling this feature, developers may notice some GameObjects only render in one eye. This is because the associated custom shaders do not have the appropriate properties for instancing.
> 
> ([Performance recommendations for Unity](https://learn.microsoft.com/en-us/windows/mixed-reality/develop/unity/performance-recommendations-for-unity?tabs=openxr)からの引用)

つまりユーザが作成したシェーダーがインスタンス化に対応していない場合，一部のObjectが片方の目でのみレンダリングされる可能性があることが示唆されており，今回の場合はこれが原因となっていると考えられます．この問題の対処法として[一部のwebサイト](https://zenn.dev/dami/articles/8f436e67cbf882)では，Stereo Rendering ModeをMulti-passに変更することで改善されたと記述されているものが見受けられました．

しかしこのStereo Rendering ModeをMulti-passにして実行しても改善されない場合もあり，それが今回に当てはまります．先ほど作成した全天球シェーダーを利用し，Stereo Rendring ModeをMulti-passにして実行した際のMeta Quest2でのスクリーンショットを下に示します．

![Quest2のスクリーンショット](Document/Screenshot_PanoramicSphere.png)

このように全天球カスタムシェーダーはインスタンス化に対応しておらず，全天球が左眼でのみ表示され右眼で表示されなくなってしまいます．
<br><br>


## 全天球シェーダー（両眼対応 Sphereモデル）
### BinocularPanoramicSphereShader.sharder

片目（左目）でのみ描画される問題は作成されたシェーダーがインスタンス化に対応していないことが原因であるため，インスタンス化に対応するようなシェーダーに変更することでこの問題は解決されます． 具体的な両眼対応の全天球シェーダーは次の手順により作成することができます．なお，以下に示す解決手法は[2018.4 Unity Documentation - Single Pass Instanced rendering](https://docs.unity3d.com/2018.4/Documentation/Manual/SinglePassInstancing.html)を参考にしたものです．

1. struct appdata{}に次の一文を追記．
    - UNITY_VERTEX_INPUT_INSTANCE_ID
2. struct v2f{}の中に次の一文を追記．
    - UNITY_VERTEX_OUTPUT_STEREO
3. v2f vert(appdata v){}の中に次の三文を追記（なお追記箇所は v2f o;の直後）.
    - UNITY_SETUP_INSTANCE_ID(v);
    - UNITY_INITIALIZE_OUTPUT(v2f, o);
    - UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);

上記1～3の手順を踏んだシェーダー全体のスクリプトは次のようになります．なお，今回新たに追記した箇所にはコメントアウトで**new added**と記してあります．

```
Shader "Unlit/BinocularPanoramicSphereShader"
{
    Properties
    {
        _MainTex("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType" = "Opaque" }
        Cull Front // added
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;

                UNITY_VERTEX_INPUT_INSTANCE_ID // new added
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                UNITY_FOG_COORDS(1)
                float4 vertex : SV_POSITION;

                UNITY_VERTEX_OUTPUT_STEREO //new added
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;

            v2f vert(appdata v)
            {
                v.uv.x = 1 - v.uv.x; //added
                v2f o;
                UNITY_SETUP_INSTANCE_ID(v); // new added
                UNITY_INITIALIZE_OUTPUT(v2f, o); // new added
                UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o); // new added

                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                UNITY_TRANSFER_FOG(o,o.vertex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                // sample the texture
                fixed4 col = tex2D(_MainTex, i.uv);
                // apply fog
                UNITY_APPLY_FOG(i.fogCoord, col);
                return col;
            }
            ENDCG
        }
    }
}
```

また上記の両眼対応全天球シェーダーを利用し，Stereo Rendring ModeをMultiviewにして実行した際のMeta Quest2でのスクリーンショットを下に示します．

![Quest2のスクリーンショット](Document/Screenshot_BinocularPanoramicSphereSharder.png)

このように，全天球カスタムシェーダーをインスタンス化に対応させることで，全天球が両目で表示されるようにすることができました！シェーダーのインスタンス化については今まで意識せず，触れてこなかったモノなので，両目で表示されるようにするために少し時間がかかりました．一人でもお役に立てれば幸いです．
<br><br>


## Unity Projectの利用について
本リポジトリのUnityProjectはOculus Integration (Deprecated)を含むVRデバイス用のAssetが含まれていません．そのため，HMDの利用時はそれぞれ適切なAssetをPackage Managerからインストールしてください．Quest利用時はUnity Projectを起動した後に，[こちらのページ](https://assetstore.unity.com/packages/tools/integration/oculus-integration-deprecated-82022)からOculus Integrationをダウンロードを行い，ご利用をお願い致します．なお，本ページに記載されているUnity ProjectのEditor Versionは2022.3.28f1です．

また，MainSceneはWorkSpace > Sceneの階層にあり，本Projectで作成したSharderはWorkSpace > Sharderの階層に配置しております．

> [!Note]
> HMDで実行する際は，File > Bulid Settings...を選択後，適切なPlatformが選択されていることを確認してからの実行をお願いします．
