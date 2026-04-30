# End-to-End Vision-to-PINN Pipeline for Structural Health Monitoring

> **Status: Pre-Publication / Active Research**
> Please note: The source code for this project is currently withheld pending publication and approval from the primary investigator. This repository serves as a technical overview of the system architecture, mathematical formulations, and finite-element validation results.

## Abstract
This project presents an automated structural assessment system that couples a Computer Vision (CV) geometry extraction pipeline with a mesh-free Physics-Informed Neural Network (PINN) computational backend. 

Designed for Mode I fracture analysis in concrete infrastructure, the system translates a single field photograph into a direct structural safety margin ($K_I / K_{Ic}$). It eliminates the need for manual measurement, mesh generation, or pre-computed FEM surrogate databases.

---

## System Architecture for PINN

<img width="1457" height="436" alt="simple_architecture_flowchart" src="https://github.com/user-attachments/assets/63e1425f-1213-411b-b5cb-12011b952c7c" />

Note: The complete pipeline operates in two primary stages: geometry extraction via CV, and linear elastic PDE optimization via PINN. The PINN part is being depicted here.

---

## Stage 1: Computer Vision Geometry Engine
The CV pipeline acts as a filter, running in five stages to ensure the physics backend only receives structurally significant, Mode I-aligned crack data.

1. **Image Preparation:** The field photograph is converted to grayscale and normalized to eliminate uneven lighting and shadow artifacts.
2. **Deep Learning Segmentation:** Utilizes a pixel-level deep learning model trained on concrete imagery to extract binary masks of the crack topology, avoiding the inaccuracy of standard bounding boxes.
3. **Structural Significance Filter:** Automatically computes mean crack width and discards instances below the American Concrete Institute 0.3 mm threshold, filtering out non-structural damage.
4. **Geometric & Structural Analysis:**
   * **Skeletonization:** Thins the mask to a 1-pixel centerline to extract exact $(a/W)$ endpoint coordinates.
   * **Width Distribution:** Utilizes a distance transform to map the full width profile along the crack.
   * **Load Alignment:** Calculates the dominant orientation and compares it to user-specified applied loads to isolate pure Mode I tensile cracks.
5. **Fourier Domain Analysis:** Transforms the spatial mask into the frequency domain (2D FFT) to reveal structural data hidden from spatial geometry.
   * **Power Spectral Density (PSD):** The 1D power curve slope is computed to quantify fracture roughness and damage accumulation.
   * **Equivalent Ellipse:** An ellipse is fitted to the central frequency region. Its aspect ratio mathematically verifies single-direction propagation versus multi-branching.

---

## Stage 2: PINN Architecture
Standard Multi-Layer Perceptrons (MLPs) suffer from spectral bias, rendering them incapable of resolving the infinite stress gradients at a crack tip. To bypass this, we architected a PINN with embedded Williams enrichment.

### Governing & Constitutive Equations
The network is trained to satisfy the Cauchy momentum equations for linear elasticity. For a domain without body forces, the equilibrium equations are:
$$\frac{\partial \sigma_{xx}}{\partial x} + \frac{\partial \sigma_{xy}}{\partial y} = 0$$
$$\frac{\partial \sigma_{xy}}{\partial x} + \frac{\partial \sigma_{yy}}{\partial y} = 0$$

The concrete is modeled as a homogeneous, isotropic solid ($E = 30,000$ MPa, $\nu = 0.20$). The plane strain constitutive law is applied:
$$\sigma_{xx} = \lambda\left(\frac{\partial u}{\partial x} + \frac{\partial v}{\partial y}\right) + 2\mu \frac{\partial u}{\partial x}$$
$$\sigma_{yy} = \lambda\left(\frac{\partial u}{\partial x} + \frac{\partial v}{\partial y}\right) + 2\mu \frac{\partial v}{\partial y}$$
$$\sigma_{xy} = \mu\left(\frac{\partial u}{\partial y} + \frac{\partial v}{\partial x}\right)$$

### Williams Near-Tip Expansion
To capture the singularity without spectral bias, the $O(\sqrt{r})$ Mode I leading-order displacement field derived by Williams (1957) is embedded as a hardcoded basis:
$$u_w = \sqrt{\frac{r}{2\pi}} \cos\left(\frac{\theta}{2}\right) \left(\kappa - 1 + 2\sin^2\left(\frac{\theta}{2}\right)\right)$$
$$v_w = \sqrt{\frac{r}{2\pi}} \sin\left(\frac{\theta}{2}\right) \left(\kappa + 1 - 2\cos^2\left(\frac{\theta}{2}\right)\right)$$
*(Where $\kappa = 3 - 4\nu$ for plane strain)*

### Network Architecture
The network jointly optimizes a smooth displacement field (via a 4x128 Tanh MLP) and a learnable scalar amplitude ($K_I$). 
* $u_{total} = u_{smooth}(x,y) + A \cdot w_u(r,\theta)$
* $v_{total} = v_{smooth}(x,y) + A \cdot w_v(r,\theta)$

**Key Features:**
* **Volumetric Cage (Soft-Loss Penalty):** Rather than using a hard spatial cutoff $\psi(r)$ in the forward pass, which generates fictitious body forces that violate Cauchy momentum equations, we apply a Gaussian volumetric cage as a soft penalty in the loss function. This forces the smooth MLP to zero near the tip, allowing the Williams basis to dominate the singularity while preserving perfect PDE equilibrium.
* **Positive Constrained Amplitude:** The extracted stress intensity factor is constrained via a softplus activation ($A = \frac{K_I}{2\mu} = \text{softplus}(K_{I\_raw})$) to guarantee physical positivity under tensile loading.

---

## Stage 3: FEniCS Validation
The PINN architecture was validated against both the Tada analytical handbook and a high-fidelity finite-element simulation generated via FEniCS.

### FEM Mesh & Reference Data
* **Mesh Structure:** 9,841 vertices and 19,134 triangular elements simulating an $a/W = 0.20$ SEN plate.
* **Energy Consistency:** The J-integral was evaluated across three concentric annuli ($r = [1,5], [2,9], [3,12]$ mm), yielding a path variation of exactly 0.0%, confirming absolute mesh convergence.

### Tada Handbook Reference
The analytical baseline was calculated using the Tada stress intensity handbook: $K_I = \sigma_{app} \sqrt{\pi a} F(a/W)$ (Given $\sigma_{app} = 100$ MPa, $a = 20$ mm, and $F(0.20) = 1.37$)

### Results Table (Plane Strain Formulation)
To verify self-consistency, the PINN extracted $K_I$ via two independent internal methods: the learned network parameter (Param) and a least-squares Williams displacement fit at $r = 6$ mm.

| Method | Extracted $K_I$ (MPa$\sqrt{\text{mm}}$) | vs. Tada Analytical | vs. FEniCS FEM |
| :--- | :--- | :--- | :--- |
| **FEniCS (Reference)** | 1083.75 | 0.2% | N/A |
| **PINN (Param)** | 1032.6 | 4.7% | 4.7% |
| **PINN (Williams Fit)** | 1021.5 | 5.8% | 5.8% |

The Param and Williams Fit methods agreed within 1.1%, demonstrating self-consistency of the learned displacement fields. Both formulations extracted $K_I$ within the <6% tolerance for mesh-free fracture mechanics.

---

## Future Work
* **Parametric Scaling:** Parameterize the volumetric cage radius ($R_{CUT}$) dynamically based on CV-extracted crack lengths to handle multi-scale damage.
* **Mixed-Mode Implementation:** Upgrade the Williams basis to incorporate Mode II (shear) variables for inclined crack assessment.
* **Hardware Deployment:** Finalize integration of the CV and PINN pipeline onto the NVIDIA Jetson Orin Nano for edge inference on a rover platform.
