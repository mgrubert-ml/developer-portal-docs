---

title: MLWebRTCCameraVideoSource.cs

---


# MLWebRTCCameraVideoSource.cs









## Source code

```csharp
// %BANNER_BEGIN%
// ---------------------------------------------------------------------
// %COPYRIGHT_BEGIN%
// Copyright (c) (2019-2022) Magic Leap, Inc. All Rights Reserved.
// Use of this file is governed by the Software License Agreement, located here: https://www.magicleap.com/software-license-agreement-ml2
// Terms and conditions applicable to third-party materials accompanying this distribution may also be found in the top-level NOTICE file appearing herein.
// %COPYRIGHT_END%
// ---------------------------------------------------------------------
// %BANNER_END%

using System;
using System.Collections.Generic;
using UnityEngine.XR.MagicLeap.Native;
using static UnityEngine.XR.MagicLeap.MLWebRTC.VideoSink.Frame;


namespace UnityEngine.XR.MagicLeap
{
    public partial class MLWebRTC
    {
        public class MLCameraVideoSource : AppDefinedVideoSource
        {
            private MLCameraVideoSource(MLCameraBase mlCameraBase, MLCamera.CaptureConfig camCaptureConfig, string trackId, Renderer localRenderer, bool nativeBuffers, bool handlePause)
                : base(trackId)
            {
                camera = mlCameraBase;
                captureConfig = camCaptureConfig;
                useNativeBuffers = nativeBuffers;
                previewRenderer = localRenderer;
                this.handlePause = handlePause;
                MLDevice.RegisterUpdate(Update);
            }

            ~MLCameraVideoSource()
            {
                MLDevice.UnregisterUpdate(Update);
            }

            private void CameraRecorderSurface_OnFrameAvailable()
            {
                surfaceFrameAvailable = 1;
            }


            private CircularBuffer<PlaneInfo[]> imagePlanesBuffer = CircularBuffer<PlaneInfo[]>.Create(
                new PlaneInfo[NativeImagePlanesLength[OutputFormat.YUV_420_888]],
                new PlaneInfo[NativeImagePlanesLength[OutputFormat.YUV_420_888]],
                new PlaneInfo[NativeImagePlanesLength[OutputFormat.YUV_420_888]]);

            private MLCameraBase camera;
            private MLCamera.CaptureConfig captureConfig;
            private MLNativeSurface cameraRecorderSurface;
            private NativeBufferInfo nativeBufferInfo;
            private Renderer previewRenderer;

            private bool isCapturing = false;
            private bool shouldStopCapturing = false;
            private int surfaceFrameAvailable = 0;
            private bool isCapturingPreview = false;
            private float previewAspectRatio = 0;
            private bool useNativeBuffers = true;
            private bool applicationPause = false;
            private bool destroyed = false;
            private bool handlePause = false;

            public bool IsCapturing => isCapturing;

            public delegate void CaptureStatusChangedDelegate(bool isDestroying);

            public event CaptureStatusChangedDelegate OnCaptureStatusChanged;

            private static readonly ushort captureBufferCount = 4;

            public static MLCameraVideoSource CreateLocal(MLCameraBase camera, MLCamera.CaptureConfig captureConfig, out MLResult result, string trackId = "", Renderer localRenderer = null, bool nativeBuffers = true, bool handlePause = true)
            {
                if (camera == null || captureConfig.StreamConfigs == null || captureConfig.StreamConfigs.Length < 1)
                {
                    result = MLResult.Create(MLResult.Code.InvalidParam);
                    return null;
                }

                MLCameraVideoSource mlCameraVideoSource = new MLCameraVideoSource(camera, captureConfig, trackId, localRenderer, nativeBuffers, handlePause);

                result = InitializeLocal(mlCameraVideoSource);
                if (!result.IsOk)
                {
                    Debug.LogError($"MLCameraVideoSource.InitializeLocal failed with result {result}");
                    return null;
                }

                mlCameraVideoSource.StartCapture();

                return mlCameraVideoSource;
            }

            protected override void OnSourceSetEnabled(bool enabled)
            {
                if (enabled)
                {
                    StartCapture();
                }
                else
                {
                    StopCapture();
                }
            }

            protected override void OnSourceDestroy()
            {
                destroyed = true;
                StopCapture();
            }

            private void ConfigureRecorderSurface()
            {
                if (!useNativeBuffers)
                {
                    return;
                }

                cameraRecorderSurface = new MLNativeSurface(MLNativeSurface.PixelFormat.Rgba8888, captureBufferCount, (uint)captureConfig.StreamConfigs[0].Width, (uint)captureConfig.StreamConfigs[0].Height);
                cameraRecorderSurface.OnFrameAvailable += CameraRecorderSurface_OnFrameAvailable;

                this.captureConfig.StreamConfigs[0].Surface = cameraRecorderSurface;

                nativeBufferInfo = new NativeBufferInfo();
                nativeBufferInfo.Width = (uint)captureConfig.StreamConfigs[0].Width;
                nativeBufferInfo.Height = (uint)captureConfig.StreamConfigs[0].Height;
                nativeBufferInfo.SurfaceHandle = cameraRecorderSurface.Handle;
                nativeBufferInfo.Transform = new float[16];
                // Initialize with identity matrix incase native call fails and we still end up sending this transform to the shaders.
                Array.Copy(Native.MLConvert.IdentityMatrixColMajor, nativeBufferInfo.Transform, nativeBufferInfo.Transform.Length);

                // The transform matrix we recieve from the underlying stack for "virtual only" camera
                // is incorrect and rotates the texture by 90 degrees instead of flipping it vertically
                // & horizontally. So just for this particular case we use our own matrix i.e.
                // an identity matrix flipped vertically, then horizontally.
                if (camera.ConnectionContext.Flags == MLCamera.ConnectFlag.VirtualOnly)
                {
                    Native.MLConvert.FlipTransformMatrixVertically(nativeBufferInfo.Transform);
                    Native.MLConvert.FlipTransformMatrixHorizontally(nativeBufferInfo.Transform);
                }
            }

            private void AdjustPreviewDimensions(float textureWidth, float textureHeight, Renderer renderer)
            {
                if (textureHeight == 0)
                {
                    Debug.LogError("AdjustPreviewDimensions can't scale renderer because it received a textureHeight of 0");
                    return;
                }

                float ratio = textureWidth / textureHeight;

                if (Math.Abs(previewAspectRatio - ratio) < float.Epsilon)
                    return;

                previewAspectRatio = ratio;
                var localScale = renderer.transform.localScale;
                localScale = new Vector3(previewAspectRatio * localScale.y, localScale.y, 1);
                renderer.transform.localScale = localScale;
            }

            private void StartCapture()
            {
                if (!isCapturing)
                {
                    Debug.Log("StartCapture with stream config: " + captureConfig.StreamConfigs[0]);

                    if (useNativeBuffers)
                    {
                        ConfigureRecorderSurface();
                    }

                    MLResult result = camera.PrepareCapture(captureConfig, cameraMetadata: out _);
                    if (result.IsOk)
                    {
                         camera.PreCaptureAEAWB();
                        if (captureConfig.StreamConfigs.Length == 2 && captureConfig.StreamConfigs[1].CaptureType == MLCamera.CaptureType.Preview)
                        {
                            result = camera.CapturePreviewStart();
                            if (result.IsOk)
                            {
                                isCapturingPreview = true;
                            }
                        }
                        if (isCapturingPreview && previewRenderer != null)
                        {
                            AdjustPreviewDimensions(camera.PreviewTexture.width, camera.PreviewTexture.height, previewRenderer);
                            previewRenderer.enabled = true;
                            previewRenderer.material.mainTexture = camera.PreviewTexture;
                        }
                        result = camera.CaptureVideoStart();
                        if (!result.IsOk)
                        {
                            Debug.LogError("capture start error: " + result);
                        }
                    }
                    else
                    {
                        Debug.LogError("prepare capture start error: " + result);
                    }
                    isCapturing = result.IsOk;
                    if (IsCapturing)
                    {
                        SetupCameraCallbacks();
                        OnCaptureStatusChanged?.Invoke(false);
                    }
                }
            }

            private void StopCapture()
            {
                if (isCapturing && !applicationPause)
                {
                    shouldStopCapturing = true;
                    if (!useNativeBuffers)
                    {
                        isCapturing = false;
                        TerminateCaptureNow();
                    }
                }
            }

            private void SetupCameraCallbacks()
            {
                if (camera != null)
                {
                    if (!useNativeBuffers)
                    {
                        camera.OnRawVideoFrameAvailable_NativeCallbackThread += Camera_OnRawVideoFrameAvailable_NativeCallbackThread;
                    }
                    camera.OnCaptureAborted += Camera_OnCaptureAborted;
                    camera.OnCaptureFailed += Camera_OnCaptureFailed;
                    camera.OnDeviceDisconnected += Camera_OnDeviceDisconnected;
                    camera.OnDeviceIdle += Camera_OnDeviceIdle;
                    camera.OnDeviceError += Camera_OnDeviceError;
                    camera.OnDeviceStreaming += Camera_OnDeviceStreaming;
                }
            }

            private void RemoveCameraCallbacks()
            {
                if (camera != null)
                {
                    if (!useNativeBuffers)
                    {
                        camera.OnRawVideoFrameAvailable_NativeCallbackThread -= Camera_OnRawVideoFrameAvailable_NativeCallbackThread;
                    }
                    camera.OnCaptureAborted -= Camera_OnCaptureAborted;
                    camera.OnCaptureFailed -= Camera_OnCaptureFailed;
                    camera.OnDeviceDisconnected -= Camera_OnDeviceDisconnected;
                    camera.OnDeviceIdle -= Camera_OnDeviceIdle;
                    camera.OnDeviceError -= Camera_OnDeviceError;
                    camera.OnDeviceStreaming -= Camera_OnDeviceStreaming;
                }
            }

            protected override void OnApplicationPause(bool pause)
            {
                if (handlePause)
                {
                    applicationPause = pause;
                    base.OnApplicationPause(pause);
                }
            }

            private void Update()
            {
                if (isCapturing)
                {
                    if (useNativeBuffers)
                    {
                        // This needs to be an atomic operation because CameraRecorderSurface_OnFrameAvailable is called on a different thread than this func.
                        bool acquireNewFrame = (System.Threading.Interlocked.Exchange(ref surfaceFrameAvailable, 0) == 1);

                        if (acquireNewFrame)
                        {
                            cameraRecorderSurface.GetFrameNumber(out ulong frameId);
                            cameraRecorderSurface.GetFrameTimestamp(out long timestampNs);
                            // The transform matrix we recieve from the underlying stack for "virtual only" camera
                            // is incorrect and rotates the texture by 90 degrees instead of flipping it vertically
                            // & horizontally. So just for this particular case we use our own matrix i.e.
                            // an identity matrix flipped vertically, then horizontally which is setup at the
                            // ConfigureRecorderSurface() function.
                            if (camera.ConnectionContext.Flags != MLCamera.ConnectFlag.VirtualOnly)
                            {
                                cameraRecorderSurface.GetFrameTransformMatrix(nativeBufferInfo.Transform);
                            }
                            // DO NOT release the frame acquired here. It will be done by the underlying webrtc lib.
                            if (cameraRecorderSurface.AcquireNextAvailableFrame(out ulong nativeBufferHandle).IsOk)
                            {
                                nativeBufferInfo.NativeBufferHandle = nativeBufferHandle;
                                VideoSink.Frame frame = VideoSink.Frame.Create(frameId, (ulong)timestampNs / 1000, nativeBufferInfo);
                                PushFrame(frame);
                            }
                        }

                        if (shouldStopCapturing)
                        {
                            TerminateCaptureNow();
                            return;
                        }
                    }
                }
            }

            private void TerminateCaptureNow()
            {
                isCapturing = false;
                if (useNativeBuffers)
                {
                    cameraRecorderSurface.OnFrameAvailable -= CameraRecorderSurface_OnFrameAvailable;
                }

                 camera.CaptureVideoStop();
                if (isCapturingPreview)
                {
                     camera.CapturePreviewStop();
                    isCapturingPreview = false;
                }
                shouldStopCapturing = false;
                
                RemoveCameraCallbacks();
                cameraRecorderSurface = null;
                captureConfig.StreamConfigs[0].Surface = null;
                OnCaptureStatusChanged?.Invoke(destroyed);
            }

            private void Camera_OnRawVideoFrameAvailable_NativeCallbackThread(MLCamera.CameraOutput cameraOutput, MLCamera.ResultExtras results, MLCamera.Metadata metadataHandle)
            {
                if (useNativeBuffers || !isCapturing)
                {
                    return;
                }

                PlaneInfo[] imagePlaneArray = imagePlanesBuffer.Get();
                for (int i = 0; i < cameraOutput.Planes.Length; ++i)
                {
                    imagePlaneArray[i] = PlaneInfo.Create(cameraOutput.Planes[i].Width, cameraOutput.Planes[i].Height, cameraOutput.Planes[i].Stride, cameraOutput.Planes[i].BytesPerPixel, cameraOutput.Planes[i].Size, cameraOutput.Planes[i].DataPtr);
                }

                OutputFormat outFmt = OutputFormat.YUV_420_888;
                if (cameraOutput.Format == MLCamera.OutputFormat.RGBA_8888)
                {
                    outFmt = OutputFormat.RGBA_8888;
                }

                MLTime.ConvertMLTimeToSystemTime(results.VCamTimestamp, out long nanoseconds);

                var frame = Create((ulong)results.FrameNumber, (ulong)nanoseconds / 1000, imagePlaneArray, outFmt);

                PushFrame(frame);
            }

            private void Camera_OnDeviceError(MLCamera.ErrorType error)
            {
                MLPluginLog.Error($"MLWebRTC.CameraVideoSource camera device error: {error}");
            }

            private void Camera_OnDeviceIdle()
            {
                MLPluginLog.Debug("MLWebRTC.CameraVideoSource camera is idle");
            }

            private void Camera_OnDeviceDisconnected(MLCamera.DisconnectReason reason)
            {
                MLPluginLog.Debug($"MLWebRTC.CameraVideoSource camera disconnected. reason: {reason}");
            }

            private void Camera_OnCaptureFailed(MLCamera.ResultExtras extra)
            {
                MLPluginLog.Error($"MLWebRTC.CameraVideoSource camera capture failed [frame: {extra.FrameNumber} timestamp: {extra.VCamTimestamp}]");
            }

            private void Camera_OnCaptureAborted()
            {
                MLPluginLog.Error($"MLWebRTC.CameraVideoSource camera capture aborted");
            }

            private void Camera_OnDeviceStreaming()
            {
                MLPluginLog.Debug("MLWebRTC.CameraVideoSource camera is streaming.");
            }
            public MLCameraVideoSource(string trackId) : base(trackId)
            {
            }
        }
    }
}
```




