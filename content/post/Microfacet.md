---
date: 2020-03-21
title: Microfacet Reflection Model
tags: ["Computer Graphics", "Notes", "Physically-Based Rendering"]
--- 
{{< figure library="true" src="microfacetReflection.png" title="Figure 1. Model rendered with microfacet reflection BRDF" lightbox="true" >}}

### 1. Overview
To get a more realistic image, many geometric-optics-based approach modeling surface reflection and transmission based on the idea that rough surface can be modeled as a collection of small microfacet, and describe their orientation (normal vectors) statistically.

{{< figure library="true" src="microfacetSurface.png" title="Figure 2. Microfacet models are often described by a function that gives the distribution of microfacet normals **$n_f$** with respect to surface normal **n**. " lightbox="true" >}}

To compute the reflection of such models, we also need to consider some local lighting effects. 
{{< figure library="true" src="localEffects.png" title="Figure 3. Three important geometric effects to consider with microfacet reflection models. (a) _Masking_: the microfacet of interest isn't visible to the viewer due to the occlusion. (b) _Shadowing_: light doesn't reach the microfacet. (c) _Interreflection_: light bounces among the microfacets before reaching the viewer." lightbox="true" >}}

Different microfacet-based [BRDF](https://en.wikipedia.org/wiki/Bidirectional_reflectance_distribution_function) (Bidirectional Reflectance Distribution Function) models consider/estimate each of these effects with varying degrees of accuracy. The general approach is to make the best approximations possible, while still obtaining an easily evaluated expression.

### 2. Oren-Nayar Diffuse Reflection 
Oren and Nayar figured that real world objects do not exhibit perfect [Lambertian Refelction](https://en.wikipedia.org/wiki/Lambert%27s_cosine_law), and they developed a reflection model that describes rough surfaces by V-shaped microfacets described by a spherical Gaussian distribution with a single parameter $\sigma$, the standard deviation of the microfacet orientation angle. The resulting model doesn't have a close-form solution, they approxiamted with:
$$BRDF: f_r(\omega_i, \omega_o) = \frac{R}{\pi}(A + Bmax(0, cos (\phi_i - \phi_o)) sin\alpha tan\beta)$$
where if $\sigma$ is in radians,
$$A = 1 - \frac{\sigma^{2}}{2(\sigma^{2} + 0.33)}$$
$$B = \frac{0.45\sigma^{2}}{\sigma^{2} + 0.99}$$
$$\alpha = max(\theta_i, \theta_o)$$
$$\beta = min(\theta_i, \theta_o)$$

### 3. Microfacet Distribution Functions

Reflection models based on microfacets that exhibit perfect specular reflection and transmission have been effective at modeling light scattering from a variety of glossy materials, including metals, plastic, and frosted glass. A common interface in renderer to reresent microfacet models could be like this:

```cpp
class MicrofacetDistribution{
	float D(const Vector3 &wh);
	float Lambda(const Vector3 &w);
	float G1(const Vector3 &w){
		return 1/(1 + Lambda(w));
	}
	float G (const Vector3 &wo, const Vector3 &wi){
		return 1/(1 + Lambda(wo) + Lambda(wi));
	}
	Vector3 Sample_wh(cosnt Vector3 &wo, const Vector2 &u);
	float Pdf (const Vector3 &wo, const Vector3 &wh);
};
```

One important characteristics of a microfacet surface is represented by the distribution function $D(\omega_h)$, which gives the differential area of microfacets with surface normal $\omega_h$. Microfacet distribution function must be normalized to ensure that they are physically plausible. More formally, given a differential area of the microsurface, dA, then the projected area of the microfacet faces above that area must be equal to dA, mathmatically defined as:
$$\int_{H^{2}(n)} D(\omega_h)cos\theta_hd\omega_h = 1$$

{{< figure library="true" src="dA.png" title="Figure 4. Given a differential area dA on a surface , then the microfacet normal distribution function  $D(\omega_h)$ must be normalized such that the projected surface area of the microfacets above the area is equal to dA." lightbox="true" >}}
The method MicrofacetDistribution::D() corresponds to the function $D(\omega_h)$; implementations return the differential area of microfacets oriented with the given normal vector $\omega_h$.
A widely used microfacet distribution function based on a Gaussian distribution of microfacet slopes is due to Beckmann and Spizzichino (1963).
The definition of Beckmann-Speizzichino model is:
$$D(\omega_h) = \frac{
						e^{-\frac{tan^2\theta_h}{\alpha^2}}
					  }
					  {\pi\alpha^2cos^4\theta_h}$$

where if \sigma is the [RMS](https://en.wikipedia.org/wiki/Root_mean_square) slope of the microfacets, the $\alpha = \sqrt{2}\sigma$.  
There is also a corresponding anisotropic version for this, simply skip it here.  
It can be convenient to specify the BRDF’s roughness with a scalar parameter in [0,1], makes the artists easily to tune the roughness during rendering. Therefore in there is usually another function coverts roughness value to $\alpha$ parameter.

### 4.Masking and Shadowing
Masking and shadowing function $G_1(\omega, \omega_h)$, gives the fraction of microfacets with normal $\omega_h$ that are visible from direction $\omega$.  
The normalization constraint for $G_1$ is:
$$ cos\theta = \int_{H^2(n)} G_1(\omega, \omega_h) max(0, \omega\cdot\omega_h)D(\omega_h)d\omega_h$$

{{< figure library="true" src="G1.png" title="Figure 5. As seen from a viewer or a light source, a differential area on the surface has area dAcos$\theta$ , where $cos\theta$ is the angle of the incident direction with the surface normal. The projected surface area of visible microfacets (thick lines) must be equal to $dAcos\theta$ as well; the masking-shadowing function  $G_1$ gives the fraction of the total microfacet area over dA that is visible in the given direction." lightbox="true" >}}

When implementing this use the notation  
- $A^+(\omega)$: projected area of forward-facing microfacet seen from $\omega$.
- $A^-(\omega)$: projected area of backward-facing microfacet seen from $\omega$.
Thus:
$$G_1(\omega) = \frac{A^+(\omega) - A^-(\omega)}{A^+(\omega)} $$
Also use another auxiliary function $\Lambda(\omega)$, corresponds to the Lambda function in the code snippet.

$$\Lambda(\omega) = \frac{A^-(\omega)}{A^+(\omega) - A^-(\omega)}  = \frac{A^-(\omega)}{cos\theta} $$
Simpliy to:
$$G_1(\omega) = \frac{1}{1 + \Lambda(\omega)}$$ 
Under the assumption of no correlation of the heights of nearby points $\Lambda(\omega)$ for the isotropic Beckmann-Spizzichino distribution is: 
$$\Lambda(\omega) = \frac{1}{2}(erf(a) -1 + \frac{e^{-a^2}}{a\sqrt{\pi}})$$
where $ a =\frac{1}{\alpha tan\theta}$ and erf is error function:
$$ erf(x) = \frac{2}{\sqrt{\pi}\int_{0}^{x}e^{-x\prime^2}dx\prime}$$ 

### 5. The Torrance-Sparrow Model
An early microfacet model was developed by Torrance and Sparrow (1967) to model metallic surfaces. They modeled surfaces as collections of perfectly smooth mirrored microfacets. Because the microfacets are perfectly specular, only those with a normal equal to the half-angle vector cause perfect specular reflection.  
Skp some derivation, we can get the Torrance-Sparrow BRDF:
$$ f_r(\omega_o, \omega_i) = \frac{D(\omega_h)G(\omega_o, \omega_i)F_r(\omega_o)}{4cos\theta_o cos\theta_i}$$

With this BRDF, we can render the microfacet reflection model objects such as figure 1.

[To be added for more detials...]
### Reference
1. [Physically Based Rendering Third Edition](http://www.pbr-book.org/3ed-2018/contents.html)