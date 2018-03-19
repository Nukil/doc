# 格灵深瞳人体算法AVX加速优化性能测试

标签（空格分隔）： Ignite

*@王振 @李坤*

[TOC]

## 性能优化目标

使用`C++`调用`AVX`指令集，在正常使用格灵深瞳算法（传入两个`byte[]`，每4位转为一个`float`，对应位的`float`相乘再进行累减）的性能基础上，比对速度提升8倍。附原始格灵深瞳`Java`版算法：

```java
public static float verify(byte[] feat1, int offset1, byte[] feat2, int offset2) {
        float score = 0.0f;
        if (feat1.length < FLOAT_SIZE * FEAT_RAW_DIM || feat2.length < FLOAT_SIZE * FEAT_RAW_DIM) {
            return score;
        }
  		// 以下为优化的关键部分
        for (int i = 0; i < FEAT_RAW_DIM; ++i) {
            float first = byte2float(feat1, i * FLOAT_SIZE + offset1);
            float second = byte2float(feat2, i * FLOAT_SIZE + offset2);
            score -= first * second;
        }

        return (float)(0.5 - 0.5 * score) * 100;
    }
```

## 直接使用AVX加速

- 普通方式进行特征比对

```cpp
float Verify(char *feat1, int offset1, char *feat2, int offset2) 
{
	float score = 0.0f;
	for (int i = 0; i < FEAT_RAW_DIM; ++i) 
	{
		float first = byte2float(feat1, i * FLOAT_SIZE);
		float second = byte2float(feat2, i * FLOAT_SIZE);
         score -= first * second;
	}
	return (0.5 - 0.5 * score) * 100;
}
```

- 使用AVX加速比对

```CPP
float VerifyByAVX(char *feat1, int offset1, char *feat2, int offset2) 
{
	float score = 0.0f;
	__m256 ymm0, ymm1, ymm2, ymm3, ymm4, ymm5, ymm6, ymm7, ymm8, ymm9, ymm10, ymm11, ymm12, ymm13, ymm14, ymm15, ymmSum;
	float fFeature1[FEAT_RAW_DIM];
	float fFeature2[FEAT_RAW_DIM];
	float iSum[8] = {0};
	
	for (int i = 0; i < FEAT_RAW_DIM; ++i) 
	{
		fFeature1[i] = byte2float(feat1, i * FLOAT_SIZE);
		fFeature2[i] = byte2float(feat2, i * FLOAT_SIZE);
	}
	
	ymmSum = __builtin_ia32_loadups256(iSum);
	
	for (int i = 0; i < FEAT_RAW_DIM; i+=64)
	{
		ymm0 = __builtin_ia32_loadups256(&fFeature1[i]);
		ymm1 = __builtin_ia32_loadups256(&fFeature1[i+8]);
 		ymm2 = __builtin_ia32_loadups256(&fFeature1[i+16]);
		ymm3 = __builtin_ia32_loadups256(&fFeature1[i+24]); 
      	 ymm4 = __builtin_ia32_loadups256(&fFeature1[i+32]);
		ymm5 = __builtin_ia32_loadups256(&fFeature1[i+40]);
        ymm6 = __builtin_ia32_loadups256(&fFeature1[i+48]);
		ymm7 = __builtin_ia32_loadups256(&fFeature1[i+56]);
        
        ymm8 = __builtin_ia32_loadups256(&fFeature2[i]);
		ymm9 = __builtin_ia32_loadups256(&fFeature2[i+8]);
        ymm10 = __builtin_ia32_loadups256(&fFeature2[i+16]);
      	ymm11 = __builtin_ia32_loadups256(&fFeature2[i+24]);
        ymm12 = __builtin_ia32_loadups256(&fFeature2[i+32]);
	    ymm13 = __builtin_ia32_loadups256(&fFeature2[i+40]);
        ymm14 = __builtin_ia32_loadups256(&fFeature2[i+48]);
        ymm15 = __builtin_ia32_loadups256(&fFeature2[i+56]);
        
        ymm0 = __builtin_ia32_mulps256(ymm0, ymm8);
        ymm1 = __builtin_ia32_mulps256(ymm1, ymm9);
        ymm2 = __builtin_ia32_mulps256(ymm2, ymm10);
        ymm3 = __builtin_ia32_mulps256(ymm3, ymm11);
        ymm4 = __builtin_ia32_mulps256(ymm4, ymm12);
        ymm5 = __builtin_ia32_mulps256(ymm5, ymm13);
        ymm6 = __builtin_ia32_mulps256(ymm6, ymm14);
        ymm7 = __builtin_ia32_mulps256(ymm7, ymm15);

        ymm0 = __builtin_ia32_addps256(ymm0, ymm1);
        ymm2 = __builtin_ia32_addps256(ymm2, ymm3);
        ymm4 = __builtin_ia32_addps256(ymm4, ymm5);
        ymm6 = __builtin_ia32_addps256(ymm6, ymm7);
        ymm0 = __builtin_ia32_addps256(ymm0, ymm2);
        ymm4 = __builtin_ia32_addps256(ymm4, ymm6);
        ymm0 = __builtin_ia32_addps256(ymm0, ymm4);

		ymmSum = __builtin_ia32_addps256(ymmSum, ymm0);
	}
	
	__builtin_ia32_storeups256(iSum, ymmSum);
	
	for (int i = 0; i < 8; ++i)
    {
		score -= iSum[i];
    }
	
	return (0.5-0.5 * score) * 100;
}
```

- 单线程100 W次特征比对测试结果（使用`GCC`编译器`-O3`优化）

  普通方式：920 ms

  AVX加速：650 ms

这种方式存在很明显的问题，每次都会把待比对的特征传入算法，进行移位运算，这里可以将待比对特征提前进行移位计算，每次传入`float[]`。

## 将待比对特征转为float[]

- 普通方式特征比对

```CPP
float Verify(float *feat1, int offset1, char *feat2, int offset2)
{
	float score = 0.0f;
	for (int i = 0; i < FEAT_RAW_DIM; ++i) 
	{
		float second = Byte2Float(feat2, i * FLOAT_SIZE);
        score -= feat1[i] * second;
	}
	return (0.5-0.5 * score) * 100;
}
```

- 使用AVX加速特征比对

```CPP
float VerifyByAVX(float *feat1, int offset1, char *feat2, int offset2)
{
	float score = 0.0f;
	__m256 ymm0, ymm1,ymm2, ymm3, ymm4, ymm5, ymm6, ymm7, ymm8, ymm9, ymm10, ymm11, ymm12, ymm13, ymm14, ymm15, ymmSum;
	float fFeature[FEAT_RAW_DIM];
	float iSum[8] = {0};
	
	for (int i = 0; i < FEAT_RAW_DIM; ++i) 
	{
		fFeature[i] = Byte2Float(feat2, i * FLOAT_SIZE);
	}
	
	ymmSum = __builtin_ia32_loadups256(iSum);
	
	for (int i = 0; i < FEAT_RAW_DIM; i+=64)
	{
		ymm0 = __builtin_ia32_loadups256(&feat1[i]);
		ymm1 = __builtin_ia32_loadups256(&feat1[i+8]);
        ymm2 = __builtin_ia32_loadups256(&feat1[i+16]);
		ymm3 = __builtin_ia32_loadups256(&feat1[i+24]);
        ymm4 = __builtin_ia32_loadups256(&feat1[i+32]);
		ymm5 = __builtin_ia32_loadups256(&feat1[i+40]); 
        ymm6 = __builtin_ia32_loadups256(&feat1[i+48]); 
		ymm7 = __builtin_ia32_loadups256(&feat1[i+56]); 
        
        ymm8 = __builtin_ia32_loadups256(&fFeature[i]);
		ymm9 = __builtin_ia32_loadups256(&fFeature[i+8]); 
        ymm10 = __builtin_ia32_loadups256(&fFeature[i+16]);
		ymm11 = __builtin_ia32_loadups256(&fFeature[i+24]);
        ymm12 = __builtin_ia32_loadups256(&fFeature[i+32]);
		ymm13 = __builtin_ia32_loadups256(&fFeature[i+40]);
        ymm14 = __builtin_ia32_loadups256(&fFeature[i+48]);
		ymm15 = __builtin_ia32_loadups256(&fFeature[i+56]);
        
        ymm0 = __builtin_ia32_mulps256(ymm0, ymm8);
        ymm1 = __builtin_ia32_mulps256(ymm1, ymm9);
        ymm2 = __builtin_ia32_mulps256(ymm2, ymm10);
        ymm3 = __builtin_ia32_mulps256(ymm3, ymm11);
        ymm4 = __builtin_ia32_mulps256(ymm4, ymm12);
        ymm5 = __builtin_ia32_mulps256(ymm5, ymm13);
        ymm6 = __builtin_ia32_mulps256(ymm6, ymm14);
        ymm7 = __builtin_ia32_mulps256(ymm7, ymm15);

        ymm0 = __builtin_ia32_addps256(ymm0, ymm1);
        ymm2 = __builtin_ia32_addps256(ymm2, ymm3);
        ymm4 = __builtin_ia32_addps256(ymm4, ymm5);
        ymm6 = __builtin_ia32_addps256(ymm6, ymm7);
        ymm0 = __builtin_ia32_addps256(ymm0, ymm2);
        ymm4 = __builtin_ia32_addps256(ymm4, ymm6);
        ymm0 = __builtin_ia32_addps256(ymm0, ymm4);

		ymmSum = __builtin_ia32_addps256(ymmSum, ymm0);
	}
	
	__builtin_ia32_storeups256(iSum, ymmSum);
	
	for (int i = 0; i < 8; ++i)
    {
		score -= iSum[i];
    }
	
	return (0.5-0.5 * score) * 100;
}
```

- 单线程100 W次特征比对测试结果（使用`GCC`编译器`-O3`优化）

  普通方式：560 ms

  AVX加速：370 ms

这种方式也存在问题，如果提前将加载到`Ignite`的特征存成`float[]`而不是`byte[]`，这样又可以节约很多移位运算时间。

## 提前将特征全部转为float[]

- 普通方式特征比对

```CPP
float verify(float *feat1, int offset1, float *feat2, int offset2) 
{
	float score = 0.0f;
	for (int i = 0; i < FEAT_RAW_DIM; ++i) 
	{
        score -= feat1[i] * feat2[i];
	}
	return (0.5-0.5 * score) * 100;
}
```

- 使用AVX加速特征比对

```CPP
 float verify_avx(float *feat1, int offset1, float *feat2, int offset2) 
{
	float score = 0.0f;
	__m256 ymm0, ymm1,ymm2, ymm3, ymm4, ymm5, ymm6, ymm7, ymm8, ymm9, ymm10, ymm11, ymm12, ymm13, ymm14, ymm15, ymmSum;
	float iSum[8] = {0};
	
	ymmSum = __builtin_ia32_loadups256(iSum);
	
	for (int i = 0; i < FEAT_RAW_DIM; i+=64)
	{
		ymm0 = __builtin_ia32_loadups256(&feat1[i]);
		ymm1 = __builtin_ia32_loadups256(&feat1[i+8]);
        ymm2 = __builtin_ia32_loadups256(&feat1[i+16]);
		ymm3 = __builtin_ia32_loadups256(&feat1[i+24]);
        ymm4 = __builtin_ia32_loadups256(&feat1[i+32]);
		ymm5 = __builtin_ia32_loadups256(&feat1[i+40]);
        ymm6 = __builtin_ia32_loadups256(&feat1[i+48]);
		ymm7 = __builtin_ia32_loadups256(&feat1[i+56]);
        
        ymm8 = __builtin_ia32_loadups256(&feat2[i]);
		ymm9 = __builtin_ia32_loadups256(&feat2[i+8]);
        ymm10 = __builtin_ia32_loadups256(&feat2[i+16]);
		ymm11 = __builtin_ia32_loadups256(&feat2[i+24]);
        ymm12 = __builtin_ia32_loadups256(&feat2[i+32]);
		ymm13 = __builtin_ia32_loadups256(&feat2[i+40]);
        ymm14 = __builtin_ia32_loadups256(&feat2[i+48]);
		ymm15 = __builtin_ia32_loadups256(&feat2[i+56]);
        
        ymm0 = __builtin_ia32_mulps256(ymm0, ymm8 );
        ymm1 = __builtin_ia32_mulps256(ymm1, ymm9 );
        ymm2 = __builtin_ia32_mulps256(ymm2, ymm10);
        ymm3 = __builtin_ia32_mulps256(ymm3, ymm11);
        ymm4 = __builtin_ia32_mulps256(ymm4, ymm12);
        ymm5 = __builtin_ia32_mulps256(ymm5, ymm13);
        ymm6 = __builtin_ia32_mulps256(ymm6, ymm14);
        ymm7 = __builtin_ia32_mulps256(ymm7, ymm15);

        ymm0 = __builtin_ia32_addps256(ymm0, ymm1);
        ymm2 = __builtin_ia32_addps256(ymm2, ymm3);
        ymm4 = __builtin_ia32_addps256(ymm4, ymm5);
        ymm6 = __builtin_ia32_addps256(ymm6, ymm7);
        ymm0 = __builtin_ia32_addps256(ymm0, ymm2);
        ymm4 = __builtin_ia32_addps256(ymm4, ymm6);
        ymm0 = __builtin_ia32_addps256(ymm0, ymm4);

        ymmSum = __builtin_ia32_addps256(ymmSum, ymm0);
        
	}
	
	__builtin_ia32_storeups256(iSum, ymmSum);
	
	for (int i = 0; i < 8; ++i)
    {
		score -= iSum[i];
    }
	
	return (0.5-0.5 * score) * 100;
}

```

- 单线程100 W次特征比对测试结果（使用`GCC`编译器`-O3`优化）

  普通方式：310 ms

  AVX加速：40 ms

使用这种方式基本达到了`AVX`加速8倍浮点数运算的效果。