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

记录当前主流图形api的常用函数及概念速查表

<!--more-->

记录当前主流图形api的常用函数及概念速查表

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

// 更新buffer数据，可初始化时更新，也可每帧更新，一般用于更新constant buffer、structure buffer、raw buffer
g_pImmediateContext->UpdateSubresource( g_pCBgpudata, 0, nullptr, &cpudata, 0, 0 );

// 用于buffer之间copy数据
g_pd3dImmediateContext->CopyResource( g_pReadBackBuffer, g_pBuffer1 );

// 创建structure buffer、raw buffer的SRV
hr = g_pd3dDevice->CreateShaderResourceView( g_pBuffer1, &srvbuffer_desc, &g_pBuffer1SRV );

// 创建structure buffer、raw buffer的UAV
hr = g_pd3dDevice->CreateUnorderedAccessView( g_pBuffer1, &uavbuffer_desc, &g_pBuffer1UAV );
```
> D3D11_BUFFER_DESC中含有Usage枚举，对应于不用cpu、gpu使用频率，合理使用；
D3D11_USAGE_DEFAULT：
A resource that requires read and write access by the GPU. This is likely to be the most common usage choice.
D3D11_USAGE_IMMUTABLE：
A resource that can only be read by the GPU. It cannot be written by the GPU, and cannot be accessed at all by the CPU. This type of resource must be initialized when it is created, since it cannot be changed after creation.
D3D11_USAGE_DYNAMIC：
A resource that is accessible by both the GPU (read only) and the CPU (write only). A dynamic resource is a good choice for a resource that will be updated by the CPU at least once per frame. To update a dynamic resource, use a Map method.
D3D11_USAGE_STAGING：
A resource that supports data transfer (copy) from the GPU to the CPU.

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
```