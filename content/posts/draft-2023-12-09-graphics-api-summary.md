---
title: 'graphics api summary'
author: wingstone
date: 2023-12-09
categories:
- rhi
- graphics api
tags: 
- 总结
draft: true
---

记录当前主流图形api的常用函数及各图形api之间的概念对比

<!--more-->

记录当前主流图形api的常用函数及各图形api之间的概念对比

## 官方demo参考

- direct3d11: https://github.com/walbourn/directx-sdk-samples
- direct3d12: https://github.com/microsoft/DirectX-Graphics-Samples
- direct3d11: https://github.com/walbourn/directx-sdk-samples

## Direct3D11

### shader编译与创建

```c++
// 从hlsl文件中编译对应shader至blob，vs、ps、cs同样适用；
hr = D3DCompileFromFile(L"VertexShader.hlsl", nullptr, nullptr, "main", "vs_5_0", D3DCOMPILE_ENABLE_STRICTNESS, 0, &vertexBlob, &errorMessage);

// 从cso文件中读取预编译好内容至blod，vs、ps、cs同样适用；
hr = D3DReadFileToBlob(L"VertexShader.cso", &vertexBlob);

// 使用blob内容创建vertex shader；
hr = _device->CreateVertexShader(vertexBlob->GetBufferPointer(), vertexBlob->GetBufferSize(), nullptr, &_vertexShader);

// 使用blob内容创建pixel shader；
hr = _device->CreatePixelShader(pixelBlob->GetBufferPointer(), pixelBlob->GetBufferSize(), nullptr, &_pixelShader);

// 使用blob内容创建compute shader；
hr = g_pd3dDevice->CreateComputeShader( pBlob->GetBufferPointer(), pBlob->GetBufferSize(), nullptr, &g_pComputeShader );
```

### 创建InputLayout

```c++
// 创建InputLayout，input layout对应vs所处理的vertex布局；
D3D11_INPUT_ELEMENT_DESC inputDesc[] =
{
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0, D3D11_INPUT_PER_VERTEX_DATA, 0 },
};
hr = _device->CreateInputLayout(inputDesc, ARRAYSIZE(inputDesc), vertexBlob->GetBufferPointer(), vertexBlob->GetBufferSize(), &_InputLayout);
```

### buffer创建

```c++
// 创建绘制所使用的buffer，vertex buffer、index buffer、constant buffer、structure rwbuffer、raw rwbuffer、readback buffer同样适用；
// InitData内含有指向cpu数据的指针，可用于创建时初始化gpu数据；
// 也可以不初始化，随后使用UpdateSubresource更新数据，不过能初始化尽量初始化
D3D11_BUFFER_DESC bufferdesc;
D3D11_SUBRESOURCE_DATA InitData；//内含有指向cpu数据的指针
hr = g_pd3dDevice->CreateBuffer( &bufferdesc, &InitData, &g_pVertexBuffer );

// 更新buffer数据 for default usage
g_pImmediateContext->UpdateSubresource( g_pCBgpudata, 0, nullptr, &cpudata, 0, 0 );

// 获取buffer数据 for staging usage，map函数还可用于dynamic usage
g_pd3dImmediateContext->Map( g_pReadBackBuffer, 0, D3D11_MAP_READ, 0, &MappedResource )
memcpy( &results[0], MappedResource.pData, NUM_ELEMENTS * sizeof(UINT) );
g_pd3dImmediateContext->Unmap( g_pReadBackBuffer, 0 );

// 用于buffer之间copy数据
g_pd3dImmediateContext->CopyResource( g_pReadBackBuffer, g_pBuffer1 );

// 创建structure buffer、raw buffer的SRV
hr = g_pd3dDevice->CreateShaderResourceView( g_pBuffer1, &srvbuffer_desc, &g_pBuffer1SRV );

// 创建structure buffer、raw buffer的UAV
hr = g_pd3dDevice->CreateUnorderedAccessView( g_pBuffer1, &uavbuffer_desc, &g_pBuffer1UAV );
```
> D3D11_BUFFER_DESC中含有Usage枚举，对应于不用cpu、gpu使用频率，合理使用；
D3D10_USAGE_DEFAULT
需要GPU的读取和写入访问权限的资源。这很可能是最常见的使用选择。常使用UpdateSubresource更新数据。 
D3D10_USAGE_IMMUTABLE
只能由GPU读取的资源。它不能由GPU写入，并且完全不能由CPU访问。此类资源必须在创建时进行初始化，因为创建后无法对其进行更改。
D3D10_USAGE_DYNAMIC
GPU和CPU都可访问的资源，(只写入)。对于每帧由CPU至少更新一次的资源，动态资源是一个不错的选择。若要写入CPU上的动态资源，请使用Map方法。可以使用 CopyResource或CopySubresourceRegion写入GPU上的动态资源。
D3D10_USAGE_STAGING
支持数据传输(将)从GPU复制到CPU的资源。
参考：[D3D11_USAGE enumeration](https://learn.microsoft.com/en-us/windows/win32/api/D3D11/ne-d3d11-d3d11_usage)以及[Copying and Accessing Resource Data (Direct3D 10)](https://learn.microsoft.com/en-us/windows/win32/direct3d10/d3d10-graphics-programming-guide-resources-mapping)


### texture创建

```c++
// 创建sampler
hr = g_pd3dDevice->CreateSamplerState( &sampDesc, &g_pSamplerLinear);

// 创建texture resource
D3D11_TEXTURE2D_DESC desc;
D3D11_SUBRESOURCE_DATA* initData;   // 与buffer创建类似，InitData内含有指向cpu texture数据的指针，可用于创建时初始化gpu texture数据；
ID3D11Texture2D* tex;
hr = d3dDevice->CreateTexture2D(&desc,
                    initData,
                    &tex
                );

// 创建贴图SRV
ID3D11Texture2D* tex;
D3D11_SHADER_RESOURCE_VIEW_DESC SRVDesc;
ID3D11ShaderResourceView* g_pTextureRV;
hr = d3dDevice->CreateShaderResourceView(tex,
            &SRVDesc,
            &g_pTextureRV
        );
```

> ID3D11Buffer与ID3D11Texture2D都继承自ID3D11Resource，两种都属于gpu资源，有一定的共性，使用ID3D11Resource作为资源参数的函数，也可作用于两者；

### Graphic管线设置

```c++
// 创建device
hr = D3D11CreateDevice( nullptr, g_driverType, nullptr, createDeviceFlags, featureLevels, numFeatureLevels,
                                D3D11_SDK_VERSION, &g_pd3dDevice, &g_featureLevel, &g_pImmediateContext );

// 创建swapchain
hr = dxgiFactory->CreateSwapChain( g_pd3dDevice, &sd, &g_pSwapChain );

// 从swapchain中获取texture
ID3D11Texture2D* pBackBuffer = nullptr;
hr = g_pSwapChain->GetBuffer( 0, __uuidof( ID3D11Texture2D ), reinterpret_cast<void**>( &pBackBuffer ) );

// 创建RTV
hr = g_pd3dDevice->CreateRenderTargetView( pBackBuffer, nullptr, &g_pRenderTargetView );

// 创建texture用于depthstencil
hr = g_pd3dDevice->CreateTexture2D( &descDepth, nullptr, &g_pDepthStencil );

// 创建CSV
hr = g_pd3dDevice->CreateDepthStencilView( g_pDepthStencil, &descDSV, &g_pDepthStencilView );

// 设置targt
g_pImmediateContext->OMSetRenderTargets( 1, &g_pRenderTargetView, g_pDepthStencilView );
       
// Setup the viewport
D3D11_VIEWPORT vp;
g_pImmediateContext->RSSetViewports( 1, &vp );

// Clear the back buffer
g_pImmediateContext->ClearRenderTargetView( g_pRenderTargetView, Colors::MidnightBlue );

// Clear the depth buffer to 1.0 (max depth)
g_pImmediateContext->ClearDepthStencilView( g_pDepthStencilView, D3D11_CLEAR_DEPTH, 1.0f, 0 );

// 设置layout布局；
g_pImmediateContext->IASetInputLayout( g_pVertexLayout );

// 设置vertex buffer；
g_pImmediateContext->IASetVertexBuffers( 0, 1, &g_pVertexBuffer, &stride, &offset );

// 设置index buffer；
g_pImmediateContext->IASetIndexBuffer( g_pIndexBuffer, DXGI_FORMAT_R16_UINT, 0 );

// 设置primitive topology（顶点buffer的几何拓扑，如果是strip，则可以省略设置index）
g_pImmediateContext->IASetPrimitiveTopology( D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST );

// 设置vs
g_pImmediateContext->VSSetShader( g_pVertexShader, nullptr, 0 );

// vs中设置const buffer
g_pImmediateContext->VSSetConstantBuffers( 0, 1, &g_pCBNeverChanges );
g_pImmediateContext->VSSetConstantBuffers( 1, 1, &g_pCBChangeOnResize );
g_pImmediateContext->VSSetConstantBuffers( 2, 1, &g_pCBChangesEveryFrame );

// 设置ps
g_pImmediateContext->PSSetShader( g_pPixelShader, nullptr, 0 );

// ps中设置const buffer
g_pImmediateContext->PSSetConstantBuffers( 2, 1, &g_pCBChangesEveryFrame );

// ps中设置srv
g_pImmediateContext->PSSetShaderResources( 0, 1, &g_pTextureRV );

// ps中设置sampler
g_pImmediateContext->PSSetSamplers( 0, 1, &g_pSamplerLinear );

// 绘制三角面数据
g_pImmediateContext->DrawIndexed( 36, 0, 0 );
void DrawIndexedInstanced(
  [in] UINT IndexCountPerInstance,
  [in] UINT InstanceCount,
  [in] UINT StartIndexLocation,
  [in] INT  BaseVertexLocation,
  [in] UINT StartInstanceLocation
);
void DrawIndexedInstancedIndirect(
  [in] ID3D11Buffer *pBufferForArgs,
  [in] UINT         AlignedByteOffsetForArgs
);

// Present our back buffer to our front buffer
g_pSwapChain->Present( 0, 0 );
```

### Compute管线设置

```c++
// cs中更新const buffer
g_pd3dImmediateContext->UpdateSubresource( g_pCB, 0, nullptr, &cb, 0, 0 );

// cs中设置const buffer
g_pd3dImmediateContext->CSSetConstantBuffers( 0, 1, &g_pCB );

// cs中设置srv（可对应buffer，也可对应texture）
g_pd3dImmediateContext->CSSetShaderResources( 0, 1, &pViewnullptr );

// cs中设置uav
g_pd3dImmediateContext->CSSetUnorderedAccessViews( 0, 1, &g_pBuffer2UAV, nullptr );

// 设置cs
g_pd3dImmediateContext->CSSetShader( g_pComputeShaderTranspose, nullptr, 0 );

// 派发cs
g_pd3dImmediateContext->Dispatch( MATRIX_WIDTH / TRANSPOSE_BLOCK_SIZE, MATRIX_HEIGHT / TRANSPOSE_BLOCK_SIZE, 1 );
void DispatchIndirect(
  [in] ID3D11Buffer *pBufferForArgs,
  [in] UINT         AlignedByteOffsetForArgs
);
```

## Direct3D12

### Graphic管线设置

```c++
// Command list allocators can only be reset when the associated 
// command lists have finished execution on the GPU; apps should use 
// fences to determine GPU execution progress.
ThrowIfFailed(m_commandAllocator->Reset());

// However, when ExecuteCommandList() is called on a particular command 
// list, that command list can then be reset at any time and must be before 
// re-recording.
ThrowIfFailed(m_commandList->Reset(m_commandAllocator.Get(), m_pipelineState.Get()));

// Set necessary state.
m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());
m_commandList->RSSetViewports(1, &m_viewport);
m_commandList->RSSetScissorRects(1, &m_scissorRect);

// Indicate that the back buffer will be used as a render target.
m_commandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(m_renderTargets[m_frameIndex].Get(), D3D12_RESOURCE_STATE_PRESENT, D3D12_RESOURCE_STATE_RENDER_TARGET));

CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHandle(m_rtvHeap->GetCPUDescriptorHandleForHeapStart(), m_frameIndex, m_rtvDescriptorSize);
m_commandList->OMSetRenderTargets(1, &rtvHandle, FALSE, nullptr);

// Record commands.
const float clearColor[] = { 0.0f, 0.2f, 0.4f, 1.0f };
m_commandList->ClearRenderTargetView(rtvHandle, clearColor, 0, nullptr);
m_commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
m_commandList->IASetVertexBuffers(0, 1, &m_vertexBufferView);
m_commandList->DrawInstanced(3, 1, 0, 0);

// Indicate that the back buffer will now be used to present.
m_commandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(m_renderTargets[m_frameIndex].Get(), D3D12_RESOURCE_STATE_RENDER_TARGET, D3D12_RESOURCE_STATE_PRESENT));

ThrowIfFailed(m_commandList->Close());
```

### 与Direct3D11的区别

#### 设备获取

Direct3D12创建设备流程如下，这是正向的设备获取流程：

1. 创建IDXGIFactory；
2. 穷举硬件设备，获取IDXGIAdapter；
3. 根据IDXGIAdapter创建ID3D12Device；

Direct3D11则采用反向的方式来获取，流程如下：

1. 将IDXGIAdapter设置为nullptr的情况下，直接创建ID3D11Device；
2. 然后获取与ID3D11Device所绑定的IDXGIAdapter；
3. 获取IDXGIAdapter所对应的parent IDXGIFactory；

也许Direct3D11也有正向获取的流程，但是官方demo里没给，毕竟没有正向获取流程，就无法人为的选择adapter；

#### 指令相关

DX12暴露了底层的command机制，所有command的执行，需要进行手动管理；其中
1. **ID3D12CommandQueue**用来执行command；
2. **ID3D12CommandAllocator**用来分配command；
3. **ID3D12GraphicsCommandList**用来记录command；
4. 三者需要在绘制之前就完成初始化；
5. 在绘制时，先进行allocator与commandlist的reset，随后进行commandlist的记录；
6. 最后使用commandqueue来执行指定的commandlist或commandlists；


#### 管线相关

dx12添加了**ID3D12PipelineState**来描述整个绘制流程所需要的状态，这些状态一次性提交给gpu，能最大化的利用gpu的性能，防止gpu处于空闲等待状态；

dx12添加了**ID3D12RootSignature**用来描述pipeline所需要绑定的资源，如sampler、srv，cbv等，graphic pipeline与compute pipeline都有自己的RootSignature；

```c++
// Describe and create the graphics pipeline state object (PSO).
D3D12_GRAPHICS_PIPELINE_STATE_DESC psoDesc = {};
psoDesc.InputLayout = { inputElementDescs, _countof(inputElementDescs) };
psoDesc.pRootSignature = m_rootSignature.Get();
psoDesc.VS = CD3DX12_SHADER_BYTECODE(vertexShader.Get());
psoDesc.PS = CD3DX12_SHADER_BYTECODE(pixelShader.Get());
psoDesc.RasterizerState = CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT);
psoDesc.BlendState = CD3DX12_BLEND_DESC(D3D12_DEFAULT);
psoDesc.DepthStencilState.DepthEnable = FALSE;
psoDesc.DepthStencilState.StencilEnable = FALSE;
psoDesc.SampleMask = UINT_MAX;
psoDesc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;
psoDesc.NumRenderTargets = 1;
psoDesc.RTVFormats[0] = DXGI_FORMAT_R8G8B8A8_UNORM;
psoDesc.SampleDesc.Count = 1;
ThrowIfFailed(m_device->CreateGraphicsPipelineState(&psoDesc, IID_PPV_ARGS(&m_pipelineState)));
}

// Create the command list.
ThrowIfFailed(m_device->CreateCommandList(0, D3D12_COMMAND_LIST_TYPE_DIRECT, m_commandAllocator.Get(), m_pipelineState.Get(), IID_PPV_ARGS(&m_commandList)));

// Command lists are created in the recording state, but there is nothing
// to record yet. The main loop expects it to be closed, so close it now.
ThrowIfFailed(m_commandList->Close());
```

#### 资源相关

在dx12中，所有的资源都使用**ID3D12Resource**来表示，不再区分buffer与texture，实际上在dx11中，buffer与texture都继承自ID3D11Resource，这里相当于抛去了这层封装；

vertexbuffer同样也由resource来表示，dx12添加了vbv（**D3D12_VERTEX_BUFFER_VIEW**）来描述vertexbuffer，类似于srv、rtv、uav；

dx12将resource区分为了CommittedResource与ReservedResource，常用的buffer与texture都属于CommittedResource，硬件VT所使用的resource才是ReservedResource；

dx12暴露了内部的**ID3D12DescriptorHeap**，来让用户手动管理Descriptor（常见的各种view的别名）；用户需要创建多个DescriptorHeap来管理不同的Descriptor；

#### 同步相关

resource的状态切换需要用户进行手动管理，需要手动调用Resource Barrier()来保证resource状态切换完毕后，才能进行后续操作；

resource数据的上传，可以通过传统map的方式上传；也可单独创建uploadbuffer并上传数据，再将数据从uploadbuffer拷贝至目标resource，随后执行所记录的command；选用哪种方式，需要根据资源cpu/gpu的使用情况来决定；

dx12暴露了内部cpu/gpu同步所使用的**ID3D12Fence**，用来保证所记录的command在当前cpu点在gpu上运行完毕；

```c++
// WAITING FOR THE FRAME TO COMPLETE BEFORE CONTINUING IS NOT BEST PRACTICE.
// This is code implemented as such for simplicity. The D3D12HelloFrameBuffering
// sample illustrates how to use fences for efficient resource usage and to
// maximize GPU utilization.

// Signal and increment the fence value.
const UINT64 fence = m_fenceValue;
ThrowIfFailed(m_commandQueue->Signal(m_fence.Get(), fence));
m_fenceValue++;

// Wait until the previous frame is finished.
if (m_fence->GetCompletedValue() < fence)
{
    ThrowIfFailed(m_fence->SetEventOnCompletion(fence, m_fenceEvent));
    WaitForSingleObject(m_fenceEvent, INFINITE);
}

m_frameIndex = m_swapChain->GetCurrentBackBufferIndex();
```

## Direct3D12与Vulkan的区别

### 设备获取

#### 概念对比

| D3D12 | Vulkan |
| --- | --- |
| debugLayer | validationLayer |
| IDXGIFactory | VkInstance |
| IDXGIAdapter | VkPhysicalDevice |
| ID3D12Device | VkDevice |

debugLayer与validationLayer都是用于使用api时，获取出错情况下的调试信息，需要在创建device时进行设置开启；

这里的命名感觉Vulkan的好些，我们后面所使用的device是抽象出来的logical device，需要与硅晶体组成的physical device进行区分；

adapter/physicaldevice对于同一台设备，可能会有多个，需要用户进行选择；

实际流程中所敲的代码，D3D12所敲的量远小于Vulkan，Vulkan的device创建过程中，需要考虑extension，Vulkan的部分功能（比如validation）需要以extension的形式开启；

Vulkan创建对象时，经常以会创建`**Info`形式的结构体来描述对象，而D3D12则是创建`**_DESC`形式的结构体来描述；

#### 创建流程

1. 创建IDXGIFactory/VkInstance，并在创建的过程中开启debugLayer/validationLayer；
2. 针对Vulkan，用于debug时，还需要创建debugMessenger（VkDebugUtilsMessengerEXT）；
3. 穷举硬件设备，获取IDXGIAdapter/VkPhysicalDevice；
4. 根据IDXGIAdapter创建ID3D12Device/VkDevice；

### 获取SwapChain

#### 概念对比

| D3D12 | Vulkan |
| --- | --- |
| HWND | VkSurfaceKHR |
| IDXGISwapChain | VkSwapchainKHR |

这里HWND与VkSurfaceKHR并不完全类似，学过windows编程的都知道，HWND代表窗口的句柄，包含了很多窗口的信息，而Vulkan的Surface代表更多的是绘制区域的含义，不管怎么，两者都能唯一包含窗口上的绘制区域；引起可以用来创建对应的SwapChain；

#### 创建流程

1. 获取HWND/VkSurfaceKHR，Vulkan可以通过GLFW提供的api获取，当不使用GLFW时，需要自己去获取；
2. 创建对应的SwapChain，D3D使用Factory进行创建，并且需要先创建CommandQueue，Vulkan直接使用Device创建即可；

### Command执行框架

#### 概念对比

| D3D12 | Vulkan |
| --- | --- |
| ID3D12CommandQueue | VkQueue |
| ID3D12CommandAllocator | VkCommandPool |
| ID3D12GraphicsCommandList | VkCommandBuffer |

这一部分命名感觉dx的更好一些：

1. CommandQueue比单纯的Queue更好的说明了其与Command相关；
2. CommandBuffer相比CommandList有一定的误导性，让人误以为其与Constant buffer等buffer一样是一种资源；

另外，Vulkan的present也需要通过Queue来执行，因此需要针对present创建单独的Queue；

#### 创建流程

1. 使用Device分别创建三个对象即可，针对Vulkan需要创建两个Queue；


### 准备RenderTarget

#### 概念对比

| D3D12 | Vulkan |
| --- | --- |
| ID3D12Resource | VkImage |
| CD3DX12_CPU_DESCRIPTOR_HANDLE | VkImageView |
|  | VkFramebuffer |

由于D3D12中Texture与其他Buffer资源的统一，所以这里的资源使用的是ID3D12Resource，而Vulkan中仍然使用VkImage与VkBuffer的区分，所以这里使用的是VkImage；

CD3DX12_CPU_DESCRIPTOR_HANDLE即在D3D中常使用的RTV，在D3D12中，所有的View都转换为了Descriptor，因此RTV这里才用了这个名字（充分说明了D3D12没有转换完整。。。），而Vulkan中依旧使用VkImageView；

Vulkan多出来了VkFramebuffer的概念，用于与RenderPass的使用相结合（我感觉VkImageView的概念与VkFramebuffer的概念有点重合。。。）；


#### 创建流程

1. 获取ID3D12Resource/VkImage，D3D12直接使用SwapChain即可获取，Vulkan需要使用Device来从SwapChain中获取；
2. 使用Device创建对应的RTV/VkImageView，因为D3D12所使用的函数仍然是`CreateRenderTargetView()`，所以我这里直接说RTV；另外由于D3D12RTV转化为了Descriptor，所以需要先创建DescriptorHeap，才能创建对应的RTV；
3. 针对Vulkan，直接使用Device创建VkFramebuffer即可；

### PipeLine资源准备

#### 概念对比

| D3D12 | Vulkan |
| --- | --- |
| ID3D12Resource | VkBuffer |
|  | VkDeviceMemory |
| D3D12_VERTEX_BUFFER_VIEW |  |
| ID3D12Resource | VkImage |
|  | VkImageView |
|  | VkSampler |
| ID3D12DescriptorHeap | VkDescriptorSet |
|  | VkDescriptorSetLayout |
|  | VkDescriptorPool |
| ID3DBlob | VkShaderModule |

Vertex Buffer/Index Buffer/Uniform Buffer都属于buffer资源，使用相同的函数创建即可
；
对于Vulkan，Buffer的存储，由单独的Memory来存储，需要进行单独创建，并绑定至对应Buffer（不如D3D，多次一举。。。）；

D3D多了VertexBufferView的概念，用于与其他View的概念对齐，Vulkan则没有此概念；因此D3D绑定VBV，Vulkan绑定Buffer（感觉对于Vulkan，Buffer成了View，Memory成了Buffer。。。）；

对于Texture，D3D仍然使用ID3D12Resource，Vulkan则使用VkImage；

D3D中ConstantBufferView的概念与ShaderResourveView的概念都转化为了Descriptor；Vulkan也使用了Descriptor的概念，但仍然保留了ImageView，不过不能直接使用，需要依托于Descriptor来使用；D3D中的Sampler不再单独创建绑定，依托于后面的ID3D12RootSignature来绑定，而Vulkan需要单独创建，依然依托于Descriptor来使用；

对于Descriptor的使用方式，Vulkan会使用VkDescriptorSetLayout来进行描述，并且此Layout用于后续绑定到PipelineLayout中；

但Descriptor并不会单独出现，其依托于ID3D12DescriptorHeap/VkDescriptorSet来使用，同时不同view的类型，需要使用多个DescriptorHeap来表示；同样，这里Vulkan多了一步，将其存储拆分到了VkDescriptorPool中；

对于buffer的同步，可以采用map/unmap以及先创建uploadbuffer，随后从uploadbuffer拷贝至目标buffer，两种类型；选用哪种方式，需要根据资源cpu/gpu的使用情况来决定；

Shader的创建，两者差别不大，使用Device创建即可；


#### 创建流程

1. 根据使用buffer的类型，使用Device进行创建即可；

### PipeLine创建与使用

#### 概念对比

| D3D12 | Vulkan |
| --- | --- |
| ID3D12RootSignature |  |
|  | VkRenderPass |
|  | VkPipelineLayout |
| ID3D12PipelineState | VkPipeline |

D3D使用ID3D12RootSignature来定义绑定到PipeLine中的资源，比如sampler；而Vulkan直接将Sampler绑定到Descriptor中；

Vulkan使用VkPipelineLayout用来将DescriptorSetLayout绑定到PipeLine中；D3D没有此环节；

Vulkan使用了RenderPass的概念来定义Pass，并且引入了SubPass的概念来面向TiledMemory；D3D12的官方Demo没有提供，但在后面的版本里也引入了RenderPass的使用；

ID3D12PipelineState/VkPipeline即我们所熟知的PSO，里面绑定了一次绘制所绑定的所有状态，包括Shader、Shader所使用的资源、BlendStat、VSLayout等等；


#### 创建流程

1. 使用Device进行创建即可；

### 帧同步

#### 概念对比

| D3D12 | Vulkan |
| --- | --- |
| ID3D12Fence | VkFence |

在Fence定义上，D3D与Vulkan并无太大差别，都是Fence配合CurrentFrameLai保证当前帧的绘制完成；


#### 创建流程

1. 使用Device进行创建即可；

