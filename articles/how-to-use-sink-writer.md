---
title: "Media FoundationのSink Writerで未圧縮の映像と音声フレームを処理して(f)MP4に変換する"
emoji: "🎬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["windows", "windowsapi", "mediafoundation", "mp4", "cpp"]
published: true
---

<!-- SPDX-License-Identifier: CC-BY-4.0 -->

## まえがき

[Sink Writer](https://learn.microsoft.com/ja-jp/windows/win32/medfound/sink-writer)は、Media Foundationで簡単に映像・音声ファイルを処理して出力できる便利なコンポーネントである。

しかし、[公式ドキュメントのチュートリアル](https://learn.microsoft.com/ja-jp/windows/win32/medfound/tutorial--using-the-sink-writer-to-encode-video)に沿ってMP4を出力しようとすると、煩瑣な違いに苛まれやすい。自力で解決しようにも、公式の情報はやや散逸しており、Webもそれに乏しい。
たとえ拙稿でも残す意義があると感じた。

以下は仮定。

* 入力はYUY2（映像）・PCM（音声…ステレオ・48kHz・16ビット）で30fps
* 出力(f)MP4はH.264（映像、QVBR）・AAC-LC（音声、CBR）で各種推奨設定を満たす
* 解像度はどちらもフルHD（1920 × 1080）
* ハードウェア処理はしない

なお、例示コードは便宜上ベタ書きであり、`HRESULT`は全て握り潰し、疑似関数用の名前空間`fakefunc`がある。

## ヘッダーの準備

### Win32 API

まず、`Windows.h`ヘッダーを取り込む。

```cpp
#define NOMINMAX
#include <Windows.h>
```

:::message alert
**直前の`NOMINMAX`マクロ定義は事実上必須。**`Windows.h`ヘッダが既定で定義する、あらゆるバグを引き起こす`min`・`max`マクロが無効化される。
:::

### Media Foundation

次に、必要なMedia Foundation関連のヘッダーを取り込む。

```cpp
#include <mfidl.h>
#include <mfapi.h>
#include <mfreadwrite.h>
```

:::message alert
**各種定義のある`mfidl.h`は他のMedia Foundation関連のヘッダーよりも先に取り込むこと。**
今回の場合、`mfreadwrite.h`が必要とするが、同ヘッダー内では取り込まれておらず、コンパイルに失敗してしまう。
:::

対応するライブラリのリンクも必要。

```cpp
#pragma comment(lib, "Mfplat")
#pragma comment(lib, "Mfreadwrite")
#pragma comment(lib, "Mfuuid")
```

### Codec API

各種設定に必要な定数のある`codecapi.h`も取り込む。

```cpp
#include <codecapi.h>
```

### Windows Implementation Library

COM用のスマートポインタを使うため、[Windows Implementation Library](https://www.nuget.org/packages/Microsoft.Windows.ImplementationLibrary)（以下、WIL）をNuGetから取得し、取り込む。

```cpp
#include <wil/com.h>
```

### 標準テンプレートライブラリ

[`std`モジュール](https://cpprefjp.github.io/module/std.html)をインポートすると、マクロを除く全ての標準テンプレートライブラリを取り込める。

```cpp
import std;
```

## Media Foundationの準備

COMを扱うため、最初に`CoInitializeEx()`を呼び、Media Foundationを扱う際は続けて`MFStartup()`も呼ぶ。`MFShutdown()`・`CoUninitialize()`で後始末。
`MFStartup()`は`MF_VERSION`マクロを渡す必要がある。

```cpp
CoInitializeEx(nullptr, COINIT_MULTITHREADED);
MFStartup(MF_VERSION);

/* 各種処理 */

MFShutdown();
CoUninitialize();
```

:::message
WILを使えば後始末を自動化できる。その場合、[WILの公式Wiki](https://github.com/microsoft/wil/wiki/RAII-resource-wrappers#wilunique_call)を参考に、自前で`MFStartup()`用の`wil::unique_call`を書く必要がある。

```cpp
// MFStartup()用は自前で書く必要がある
using unique_mfshutdown_call = wil::unique_call<decltype(&::MFShutdown), ::MFShutdown>;
[[nodiscard]] inline auto AutoMFStartup(DWORD &&flags = MFSTARTUP_FULL)
{
	::MFStartup(MF_VERSION, flags);
	return unique_mfshutdown_call();
}

{
	// CoInitializeEx()用はWILが直接提供する
	auto const com_cleanup{ wil::CoInitializeEx() };
	auto const mf_cleanup{ AutoMFStartup() };
} // ここで開放される
```
:::

以降、`CoInitializeEx()`〜`CoUninitialize()`内とする。

## Sink Writerの準備

### 空の属性ストアの準備

まず、Sink Writerの初期化時に使う、各種属性を含む`IMFAttributes`を準備する。

`IMFAttributes`を始め、Media FoundationのインターフェイスはWILのスマートポインタ`wil::com_ptr`で管理する。

```cpp
wil::com_ptr<IMFAttributes> sink_writer_attributes{};
```

準備したスマートポインタへ`MFCreateAttributes()`を呼ぶ。`MFCreateAttributes()`は第1引数に`IMFAttributes`への二重ポインタ、第2引数に初期属性数が必要。まず、属性は3つ指定する予定なため、第2引数は`3u`とする。

第1引数の渡し方は3通り。1つ目は`put()`。

```cpp
MFCreateAttributes(sink_writer_attributes.put(), 3u);
```

2つ目は`wil::com_ptr`の`&`演算子。これは前述の`put()`の略記。

```cpp
MFCreateAttributes(&sink_writer_attributes, 3u);
```

3つ目はC++23以降の標準テンプレートライブラリにある[`std::out_ptr()`](https://cpprefjp.github.io/reference/memory/out_ptr.html)。

```cpp
MFCreateAttributes(std::out_ptr(sink_writer_attributes), 3u);
```

以降、同様の処理には`std::out_ptr()`を使う。

### 属性の設定

次に、必要な属性を設定していく。

#### MF_SINK_WRITER_DISABLE_THROTTLING

既定では、Sink Writerの処理速度は意図的に下げられている。
この制限を外すのが[`MF_SINK_WRITER_DISABLE_THROTTLING`](https://learn.microsoft.com/ja-jp/windows/win32/medfound/mf-sink-writer-disable-throttling)属性。

ドキュメントにある通り、`IMFAttributes::SetUINT32()`で設定する。
なお、関数名に`UINT32`とあるが、設定する値は`Windows.h`が定義する`BOOL`型の`TRUE`。

```cpp
sink_writer_attributes->SetUINT32(MF_SINK_WRITER_DISABLE_THROTTLING, TRUE);
```

:::message
単に`true`でも可。以下、[公式ドキュメント](https://learn.microsoft.com/ja-jp/cpp/cpp/bool-cpp)より引用。

>bool 型は整数の上位変換に使用されます。 bool 型の rvalue は int 型の rvalue に変換できます。false は 0 に、true は 1 になります。
:::

#### MF_MPEG4SINK_MOOV_BEFORE_MDAT

既定では、Sink Writerが出力するMP4はデータ（以下、mdatアトム）→メタ情報（以下、moovアトム）という構造になる。
しかし、昨今のMP4は（取り分け動画サイトでは）moovアトムが最初に来るのが期待されている。
[`MF_MPEG4SINK_MOOV_BEFORE_MDAT`](https://learn.microsoft.com/ja-jp/windows/win32/medfound/mf-mpeg4sink-moov-before-mdat)属性を設定すれば、そのようなMP4を出力できる。

```cpp
sink_writer_attributes->SetUINT32(MF_MPEG4SINK_MOOV_BEFORE_MDAT, TRUE);
```

:::message
後述するが、同時に`MF_TRANSCODE_CONTAINERTYPE`を`MFTranscodeContainerType_FMPEG4`にする必要がある。
:::

#### MF_TRANSCODE_CONTAINERTYPE

後述の`MFCreateSinkWriterFromURL()`でSink Writerを作る際、拡張子が「.mp4」ならば、自動で`MFTranscodeContainerType_MPEG4`に設定される。
しかし、`MF_MPEG4SINK_MOOV_BEFORE_MDAT`を正常動作させるには、手動で`MFTranscodeContainerType_FMPEG4`に設定し、実際にはFragmented MP4を出力する必要がある。

:::message alert
`MFTranscodeContainerType_MPEG4`で`MF_MPEG4SINK_MOOV_BEFORE_MDAT`を有効にすると、**moovアトムがmdatアトムに置き換わった不正なMP4**が出力されてしまう。
:::

[`MF_TRANSCODE_CONTAINERTYPE`](https://learn.microsoft.com/ja-jp/windows/win32/medfound/mf-transcode-containertype)は`IMFAttributes::SetGUID()`で設定する。

```cpp
sink_writer_attributes->SetGUID(MF_TRANSCODE_CONTAINERTYPE, MFTranscodeContainerType_FMPEG4);
```

### Sink Writerの作成

準備した属性と共に、[`MFCreateSinkWriterFromURL()`](https://learn.microsoft.com/ja-jp/windows/win32/api/mfreadwrite/nf-mfreadwrite-mfcreatesinkwriterfromurl)でSink Writerを作る。

標準テンプレートライブラリのそれと同様に、`get()`で本体を取得可能。

```cpp
wil::com_ptr<IMFSinkWriter> sink_writer{};
MFCreateSinkWriterFromURL
(
	// 出力先（ワイド文字列、拡張子込み）
	LR"(C:\Users\Foo\output.mp4)",

	// 不使用
	nullptr,

	// 属性群へのポインタ
	sink_writer_attributes.get(),

	// Sink Writer作成先への二重ポインタ
	std::out_ptr(sink_writer)
);
```

### 入出力の設定

作ったSink Writerに映像・音声の入出力の設定をする。

#### 入力用のメディア種類の準備

まず、双方の入力用のメディア種類を準備する。

両種類群は後の処理でも使うため、取っておく。

##### 映像

[公式ドキュメント](https://learn.microsoft.com/ja-jp/windows/win32/medfound/uncompressed-video-media-types)を参考に設定していく。

まず、`IMFMediaType`を準備する。

```cpp
wil::com_ptr<IMFMediaType> input_video_media_type{};
MFCreateMediaType(std::out_ptr(input_video_media_type));
```

楽な種類から設定する。

```cpp
// 「これは映像のメディア種類です」
input_video_media_type->SetGUID(MF_MT_MAJOR_TYPE, MFMediaType_Video);

// 「非インターレースです」
input_video_media_type->SetUINT32(MF_MT_INTERLACE_MODE, MFVideoInterlace_Progressive);

// 「入力は未圧縮です」
input_video_media_type->SetUINT32(MF_MT_ALL_SAMPLES_INDEPENDENT, TRUE);
input_video_media_type->SetUINT32(MF_MT_FIXED_SIZE_SAMPLES, TRUE);
```

ヘルパー関数`MFSetAttributeSize()`で解像度を、`MFSetAttributeRatio()`でフレームレート・[ピクセル縦横比](https://helpx.adobe.com/jp/premiere/desktop/edit-projects/intro-to-editing/pixel-aspect-ratio.html)を設定する。

```cpp
MFSetAttributeSize(input_video_media_type.get(), MF_MT_FRAME_SIZE, 1920u, 1080u);

MFSetAttributeRatio(input_video_media_type.get(), MF_MT_FRAME_RATE, 30u, 1u);
MFSetAttributeRatio(input_video_media_type.get(), MF_MT_PIXEL_ASPECT_RATIO, 1u, 1u);
```

より詳細な入力種類を示す`MF_MT_SUBTYPE`の値は、予め変数に代入しておく。

```cpp
auto const video_subtype{ MFVideoFormat_YUY2 };
input_video_media_type->SetGUID(MF_MT_SUBTYPE, video_subtype);
```

この変数とヘルパー関数で、入力フレームの容量・既定ストライド値（走査線1本分の容量）を設定する。

```cpp
// 入力フレームの容量
std::uint32_t image_size{};
MFCalculateImageSize(video_subtype, 1920u, 1080u, &image_size);
input_video_media_type->SetUINT32(MF_MT_SAMPLE_SIZE, image_size);

// 既定ストライド値
// MFVideoFormat_XXX.Data1は対応するFourCC値
// MFVideoFormat_YUY2.Data1 == FCC('YUY2')
long default_stride{};
MFGetStrideForBitmapInfoHeader(video_subtype.Data1, 1920u, &default_stride);
input_video_media_type->SetUINT32(MF_MT_DEFAULT_STRIDE, default_stride);
```

:::message
既知の場合、公式では色関連の種類の設定も推奨しているが、今回は無視。
:::

##### 音声

[公式ドキュメント](https://learn.microsoft.com/ja-jp/windows/win32/medfound/uncompressed-audio-media-types)を参考。

```cpp
wil::com_ptr<IMFMediaType> input_audio_media_type{};
MFCreateMediaType(std::out_ptr(input_audio_media_type));
input_audio_media_type->SetGUID(MF_MT_MAJOR_TYPE, MFMediaType_Audio);
input_audio_media_type->SetGUID(MF_MT_SUBTYPE, MFAudioFormat_PCM);
input_audio_media_type->SetUINT32(MF_MT_AUDIO_NUM_CHANNELS, 2u);
input_audio_media_type->SetUINT32(MF_MT_AUDIO_SAMPLES_PER_SECOND, 48000u);
input_audio_media_type->SetUINT32(MF_MT_AUDIO_BITS_PER_SAMPLE, 16u);
input_audio_media_type->SetUINT32(MF_MT_ALL_SAMPLES_INDEPENDENT, TRUE);
```

:::message
「少なくとも」設定すべきとされている`MF_MT_AUDIO_BLOCK_ALIGNMENT`・`MF_MT_AUDIO_AVG_BYTES_PER_SECOND`は、MP4の場合は省略可。
:::

#### 出力ストリームの設定

準備したメディア種類で、Sink Writerの出力ストリームの設定をする。

:::message alert
**必ず**映像→音声の順番で処理する。
:::

##### 映像

[公式ドキュメント](https://learn.microsoft.com/ja-jp/windows/win32/medfound/h-264-video-encoder)を参考。

```cpp
wil::com_ptr<IMFMediaType> output_video_media_type{};
MFCreateMediaType(std::out_ptr(output_video_media_type));

// 「映像の出力設定です」
output_video_media_type->SetGUID(MF_MT_MAJOR_TYPE, MFMediaType_Video);

// 「コーデックはH.264です」
output_video_media_type->SetGUID(MF_MT_SUBTYPE, MFVideoFormat_H264);

// 「フレームレートは入力と同じです」
std::uint32_t input_video_frame_rate{}, input_video_frame_scale{};
MFGetAttributeRatio(input_video_media_type.get(), MF_MT_FRAME_RATE, &input_video_frame_rate, &input_video_frame_scale);
MFSetAttributeRatio(output_video_media_type.get(), MF_MT_FRAME_RATE, input_video_frame_rate, input_video_frame_scale);

// 「フレームの容量は入力と同じです」
std::uint32_t input_video_width{}, input_video_height{};
MFGetAttributeSize(input_video_media_type.get(), MF_MT_FRAME_SIZE, &input_video_width, &input_video_height);
MFSetAttributeSize(output_video_media_type.get(), MF_MT_FRAME_SIZE, input_video_width, input_video_height);

// 「非インターレースです」
// （≒デジタルです）
output_video_media_type->SetUINT32(MF_MT_INTERLACE_MODE, MFVideoInterlace_Progressive);

// 「Highプロファイルを使います」
// （大半の動画サイトが推奨）
output_video_media_type->SetUINT32(MF_MT_MPEG2_PROFILE, eAVEncH264VProfile_High);
```

色の情報も追加しておく。今回はFFmpegで`(tv, bt709, progressive)`と出るよう設定。

```cpp
output_video_media_type->SetUINT32(MF_MT_VIDEO_NOMINAL_RANGE, MFNominalRange_16_235);
output_video_media_type->SetUINT32(MF_MT_VIDEO_CHROMA_SITING, MFVideoChromaSubsampling_MPEG2);
output_video_media_type->SetUINT32(MF_MT_VIDEO_PRIMARIES, MFVideoPrimaries_BT709);
output_video_media_type->SetUINT32(MF_MT_YUV_MATRIX, MFVideoTransferMatrix_BT709);
output_video_media_type->SetUINT32(MF_MT_TRANSFER_FUNCTION, MFVideoTransFunc_709);
```

このメディア種類を基に、`IMFSinkWriter::AddStream()`で映像の出力ストリームを設定する。
第1引数にメディア種類へのポインタが、第2引数にストリームのインデックス番号を受け取る`DWORD`への参照が必要。

```cpp
DWORD video_index{};
sink_writer->AddStream(output_video_media_type.get(), &video_index);
```

##### 音声

[公式ドキュメント](https://learn.microsoft.com/ja-jp/windows/win32/medfound/aac-encoder)を参考。
入力用のメディア種類と一致する必要がある種類は、`MFGetAttributeUINT32()`で値を取得し、そのまま設定。第3引数は既定値。

```cpp
wil::com_ptr<IMFMediaType> output_audio_media_type{};
MFCreateMediaType(std::out_ptr(output_audio_media_type));

// 以下3つは値が決まっている
output_audio_media_type->SetGUID(MF_MT_MAJOR_TYPE, MFMediaType_Audio);
output_audio_media_type->SetGUID(MF_MT_SUBTYPE, MFAudioFormat_AAC);
output_audio_media_type->SetUINT32(MF_MT_AUDIO_BITS_PER_SAMPLE, 16u);

// 以下2つは入力用のメディア種類と一致する必要がある
output_audio_media_type->SetUINT32
(
	MF_MT_AUDIO_SAMPLES_PER_SECOND,
	MFGetAttributeUINT32(input_audio_media_type.get(), MF_MT_AUDIO_SAMPLES_PER_SECOND, 48000u)
);
output_audio_media_type->SetUINT32
(
	MF_MT_AUDIO_NUM_CHANNELS,
	MFGetAttributeUINT32(input_audio_media_type.get(), MF_MT_AUDIO_NUM_CHANNELS, 2u)
);

// 12000・16000・20000・24000しか設定できない
// この場合、それぞれ96・128・160・192kbpsに対応
output_audio_media_type->SetUINT32(MF_MT_AUDIO_AVG_BYTES_PER_SECOND, 24000u);

DWORD audio_index{};
sink_writer->AddStream(output_audio_media_type.get(), &audio_index);
```

#### 入力ストリームの設定

準備した出力ストリームのインデックス番号で、Sink Writerの入力ストリームの設定をする。

##### 映像

[公式ドキュメント](https://learn.microsoft.com/ja-jp/windows/win32/medfound/h-264-video-encoder)の「コーデックのプロパティ」は、属性として入力ストリームに設定する。

```cpp
wil::com_ptr<IMFAttributes> video_encoder_attributes{};
MFCreateAttributes(std::out_ptr(video_encoder_attributes), 4u);

// 「CABAC圧縮します」
// （大半の動画サイトが推奨）
video_encoder_attributes->SetUINT32(CODECAPI_AVEncH264CABACEnable, TRUE);

// 「Bフレーム数は2です」
// （大半の動画サイトが推奨）
video_encoder_attributes->SetUINT32(CODECAPI_AVEncMPVDefaultBPictureCount, 2u);

// 「QVBRです」
video_encoder_attributes->SetUINT32(CODECAPI_AVEncCommonRateControlMode, eAVEncCommonRateControlMode_Quality);

// 「QVBRの品質は85です」
video_encoder_attributes->SetUINT32(CODECAPI_AVEncCommonQuality, 85u);

sink_writer->SetInputMediaType(video_index, input_video_media_type.get(), video_encoder_attributes.get())
```

##### 音声

特に設定すべきものはないため、単に`IMFSinkWriter::SetInputMediaType()`を呼んで終了。

```cpp
sink_writer->SetInputMediaType(audio_index, input_audio_media_type.get(), nullptr);
```

## メイン処理

`CoInitializeEx()`のように、Sink Writerの処理は`IMFSinkWriter::BeginWriting()`〜`IMFSinkWriter::Finalize()`間で行う。

```cpp
sink_writer->BeginWriting();

/* 各種処理 */

sink_writer->Finalize();
```

Sink Writerに情報を送るには、`IMFSample`に格納してから`IMFSinkWriter::WriteSample()`で渡す。

以降、`IMFSinkWriter::BeginWriting()`〜`IMFSinkWriter::Finalize()`間で、かつ実際にはループ処理。

### 映像

まず、映像データ・再生のタイミング（以下、タイムスタンプ）は以下と仮定。

```cpp
BYTE const *raw_video_frame_begin{ fakefunc::get_raw_video_frame() };
std::int64_t video_frame_timestamp{ fakefunc::get_video_frame_timestamp() };
```

フレームレートをヘルパー関数`MFFrameRateToAverageTimePerFrame()`に渡し、サンプルの長さを求めておく。

```cpp
std::uint64_t video_sample_duration{};
MFFrameRateToAverageTimePerFrame(input_video_frame_rate, input_video_frame_scale, &video_sample_duration);
```

`IMFSample`を作るには、まず元となる`IMFMediaBuffer`を作る必要があり、公式ドキュメントのチュートリアルは`MFCreateMemoryBuffer()`で作っている。
だが、今回のような未圧縮の入力を扱う際は[`IMF2DBuffer`](https://learn.microsoft.com/ja-jp/windows/win32/api/mfobjects/nn-mfobjects-imf2dbuffer)の方がよさそうである。しかし、作り方が載っていない。

公式ドキュメントのリファレンスの方を見ていくと、[`MFCreate2DMediaBuffer()`](https://learn.microsoft.com/ja-jp/windows/win32/api/mfapi/nf-mfapi-mfcreate2dmediabuffer)が見つかる。どうやら、`IMFMediaBuffer`を渡せばそこに作ってくれるらしい。

しかし、更にリファレンスを見ていくと、[`MFCreateMediaBufferFromMediaType()`](https://learn.microsoft.com/ja-jp/windows/win32/api/mfapi/nf-mfapi-mfcreatemediabufferfrommediatype)が見つかる。渡したメディア種類に適した`IMFMediaBuffer`を作ってくれるそうで、映像ならば自動的に[`IMF2DBuffer2`](https://learn.microsoft.com/ja-jp/windows/win32/api/mfobjects/nn-mfobjects-imf2dbuffer)も作ってくれるそう。今回はこれを使うことにする。

```cpp
wil::com_ptr<IMFMediaBuffer> video_buffer{};
MFCreateMediaBufferFromMediaType
(
	// 映像入力用メディア種類
	input_video_media_type.get(),

	// サンプルの長さ
	video_sample_duration,

	// 不使用
	0, 0,

	// 映像メディアバッファーへの二重ポインタ
	std::out_ptr(video_buffer)
);
```

未圧縮の映像フレームを`IMFMediaBuffer`に渡すには、①[`IMF2DBuffer2`](https://learn.microsoft.com/ja-jp/windows/win32/api/mfobjects/nn-mfobjects-imf2dbuffer2)を問い合わせ、②書き込み専用フラグで`IMF2DBuffer2::Lock2DSize()`を呼び、③一時的に取得したポインタの先にヘルパー関数`MFCopyImage()`で書き込み、④`IMF2DBuffer2::Unlock2D()`を呼んで処理を終えればいい。

:::message
`MFCopyImage()`の代わりに`std::memcpy()`や`std::memmove()`でも可。以下、公式ドキュメントのチュートリアルより引用。
>この特定の例では、 memcpy の 使用も同様に機能します。 ただし、 MFCopyImage 関数は、ソース イメージのストライドがターゲット バッファーと一致しない場合を正しく処理します。
:::

`wil::com_ptr`は`query<IDesiredInterface>()`で簡単に問い合わせられる。

```cpp
uint8_t *scanline_begin{};
long stride{};
uint8_t *buffer_begin{};
DWORD buffer_length{};

video_buffer.query<IMF2DBuffer2>()->Lock2DSize
(
	// 書き込み専用フラグ
	MF2DBuffer_LockFlags_Write,

	// 1本目の走査線の先頭
	&scanline_begin,

	// 既定ストライド値
	&stride,

	// 不使用
	&buffer_begin, &buffer_length
);

MFCopyImage
(
	// 書込先の先頭のポインタ
	scanline_begin,

	// 書込先のストライド値
	stride,

	// 書込元の先頭のポインタ
	raw_video_frame_begin,

	// 書込元のストライド値（左）
	// ＝　書込元の走査線1本分の容量（右）
	// ＝　既定ストライド値
	stride, stride,

	// 高さ
	1080ul
);

video_buffer.query<IMF2DBuffer2>()->Unlock2D();
```

[`IMF2DBuffer`の公式ドキュメント](https://learn.microsoft.com/ja-jp/windows/win32/api/mfobjects/nn-mfobjects-imf2dbuffer)には、以下の記述がある。

>バッファー内のデータを変更する場合、サイズを更新するために`IMFMediaBuffer::SetCurrentLength`を呼び出す必要はありません。

しかし、実際には呼び出す必要がある。

```cpp
// IMF2DBuffer2の長さを取得し、
DWORD video_buffer_size{};
video_buffer.query<IMF2DBuffer2>()->GetContiguousLength(&video_buffer_size);

// IMFMediaBuffer側に伝える
video_buffer->SetCurrentLength(video_buffer_size);
```

後は`IMFSample`に格納し、Sink Writerに送るだけ。

```cpp
wil::com_ptr<IMFSample> video_sample{};
MFCreateSample(std::out_ptr(video_sample));

video_sample->AddBuffer(video_buffer.get());
video_sample->SetSampleDuration(video_sample_duration);
video_sample->SetSampleTime(video_frame_timestamp);

sink_writer->WriteSample(video_index, video_sample.get());
```

ここまでで1ループ。

### 音声

音声データ・長さ・タイムスタンプは以下と仮定。

```cpp
BYTE const *raw_audio_sample_begin{ fakefunc::get_raw_audio_sample() };
DWORD const raw_audio_sample_size{ fakefunc::get_raw_audio_sample_size() };
std::int64_t const audio_sample_duration{ fakefunc::get_audio_sample_duration() };
std::int64_t const audio_sample_timestamp{ fakefunc::get_audio_sample_timestamp() };
```

映像と同様に`MFCreateMediaBufferFromMediaType()`で`IMFMediaBuffer`を作る。

```cpp
wil::com_ptr<IMFMediaBuffer> audio_buffer{};
MFCreateMediaBufferFromMediaType
(
	input_audio_media_type.get(),
	audio_sample_duration,
	audio_sample_size, // 要求する最小サイズ
	0,
	std::out_ptr(audio_buffer)
);
```

音声は`IMFMediaBuffer`をそのまま扱う。
処理時のヘルパー関数もないため、通常通り`std::memcpy()`や`std::memmove()`、あるいは`memcpy_s()`や`memmove_s()`を使う。

```cpp
BYTE *audio_sample_begin{};
DWORD audio_data_max_size{};

audio_buffer->Lock(&audio_sample_begin, &audio_data_max_size, nullptr);

memmove_s(audio_sample_begin, audio_data_max_size, raw_audio_sample_begin, audio_sample_size);

audio_buffer->Unlock();

audio_buffer->SetCurrentLength(audio_sample_size);
```

同様に`IMFSample`に格納し、送る。

```cpp
wil::com_ptr<IMFSample> audio_sample{};
MFCreateSample(std::out_ptr(audio_sample));

audio_sample->AddBuffer(audio_buffer.get());
audio_sample->SetSampleDuration(audio_sample_duration);
audio_sample->SetSampleTime(audio_sample_timestamp);

sink_writer->WriteSample(audio_index, audio_sample.get());
```

ここまでで1ループ。

## 完了

`IMFSinkWriter::Finalize()`がサンプルを処理し、MP4を出力する。

## あとがき

後学のために[自分用のAviUtl ExEdit2の出力プラグイン](https://github.com/MonogoiNoobs/aviutl2-media-foundation-output-plugin)を書いた際の覚書。
AviUtlでの出力には[ffmpegOut](https://github.com/rigaya/ffmpegOut)を使いましょう。
