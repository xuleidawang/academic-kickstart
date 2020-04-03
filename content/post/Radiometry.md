---
date: 2020-03-30
title: My last note for Radiometry
tags: ["Computer Graphics", "Math"]
--- 
Radiometry may be one of the most troblesome topics in computer grahpics. But it is an essential concept that is the foundation for understanding BRDF and rendering equation. I've studied it for more than 5 times, but still need to find a reference when thinking about it. So I make a resolution to summary it here, and hope it is the last time.  

### 1. Basic Quantieis
There are several basic quantities people care in radiometry: **Energy**, **Flux**, **Irradiance**, **Intensity**, **Radiance**.  

#### 1.1 Radiant energy Q [J = Joule]  
It is the sources of illumination emit photons, each of which is at a particular wavelenght and carries certrain amount of energy.  
$$ Q = \frac{hc}{\lambda}$$
c is light speed, h is the Plank's constant.

#### 1.2 Radiant **Flux**(power) [J/s, W = watt]  
Flux is the total amount of energy passing through a surface of space per unit time.
Can be found by taking the limit of differential energy per differential time:
$$\Phi = \frac{dQ}{dt}$$ 

#### 1.3 **Irradiance** E [$W/m^2$]  
Power per unit area incident on a surface point p. Can defien the irradiance by taking the limit of differential power per differential area at a point p:
$$E(p) = \frac{d\Phi(p)}{dA} $$
We can also integrate over an area to find power:
$$ \Phi = \int_A E(p)dA$$

#### 1.4 Solid angle $\Omega$ or $\omega$ [steradians/sr], **Intensity** I [w/sr]  
The solid angle extends the 2D unit circle to 3D unit sphere. The total area s is the solid angle substended by the object. Measured in steradians (sr). The entire sphere subtends a solid angle of $4\pi$ sr.  Intensity is the angular density of emitted power, Intensity describes the directional distribution of light, but only meaningful for point light source.
$$ I = \frac{d\Phi}{d\omega}$$

Differential Solid Angle, It's often handy to convert the computation into spherial coordiantes, here are their relationship.
{{< figure library="true" src="DifferentialSolidAngle.png" title="*" lightbox="true" >}}

$d\omega = \frac{dA}{r^2} = sin\theta d\theta d\varphi$  
Solid angle change corrsponds to the infinitesimal change
 of $\theta$ and $\varphi$. 

#### 1.5 **Radiance** L $[\frac{W}{sr m^2}]$
Radiance measeres irradiance with respect to solid angles. Rendering is all about computing radiance.

Definition: power emitted, reflected, transmitted or received by a surface, per unit solid angle, per projected unit area.

$$L(p, \omega) = \frac{d^2\Phi(p,\omega)}{d\omega dA cos\theta}$$

**Irradiance vs. Radiance**
Irradiance: total power received by area dA  
Radiance: power received by area dA from direction $d\omega$  
$$ dE(p, \omega) = L_i(p, \omega) cos\theta d\omega$$
$$ E(p) = \int_H^2 L_i(p, \omega)cos\theta d\omega$$

### 2. BRDF and Rendering Equation
After introduced the abovementioned quantites, we are well prepared to talk about the Bidirection Reflectance Distribution Function (BRDF).

{{< figure library="true" src="reflectionAtPoint.png" title="" lightbox="true" >}}
In this figure, radiance from direction $\omega_i$ turns into the power that dA receives. Then the power will become the radiacne to any other outgoing direction $\omega_o$.  
Differential irradiance incoming: $dE(\omega_i) = L(\omega_i) cos\theta_i d\omega_i$  
Differential radiance exiting (due to $dE(\omega_i)$): $dL_r(\omega_r))$  

The BRDF represents how much light is reflected into each outgoing direction $d\omega_r$ from each incoming direction.  

{{< figure library="true" src="brdf.png" title="" lightbox="true" >}}

BRDF $$f_r(\omega_i, \omega_r) = \frac{dL_r(\omega_r)}{dE_i(\omega_i)} = \frac{dL_r(\omega_r)}{L_i(\omega_i)cos\theta_i d\omega_i}$$

And finally, we can understand the following so called rendering equation:
$$L_o(p, \omega_o) = L_e(p,\omega_o) + \int_{\Omega}f_r(p, \omega_i, \omega_o)L_i(p,\omega_i, \omega_o)(n\cdot\omega_i)d\omega_i$$