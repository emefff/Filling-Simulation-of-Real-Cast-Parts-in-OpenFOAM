# Filling-Simulation-of-Real-Cast-Parts-in-OpenFOAM

Filling simulations of cast parts are very common nowadays. Commercial packages exploiting the finite difference (FD) method exist and are readily available, although at high cost. Historically to my knowledge, all of them are based on the finite difference method due to these simulations being computationally very expensive. From the method alone it directly follows:

-) meshes are structured

-) mesh geometries are simple, mostly hex meshes are used. 

-) Mesh geometries do follow the part accurately. 'Minecrafty' look of meshes

-) implementations of turbulence models are somewhat questionable

-) mostly wall functions are used to resolve the boundary layer region

-) the package I used projects results onto the geometry. I makes the results look nicer, but does not improve results. 

Due to them being commercial packages they are:

-) closed source

-) do they even use turbulence models? (in the case of the package I used, I am not even being informed about this as a user)

-) computationally very efficient and user friendly (there's your ONE advantage)

-) VERY expensive monetarily

-) restricted, per-core licenses, which directly leads to

-) running on low core count CPUs (mostly on workstations, it directly follows that you would by it especially buy it for your 4-core or 8-core licnese), or if it this is not the case, running on servers which are probably not optimized for this kind of software. 

-) restricted regarding exporting the computed data. No, you cannot open any of these files in ParaView or VisIt.

-) post-processing is restricted to the features in the package. The formulation of own computed parameters, at least in my used package, was (it probably still is) weird, cumbersome and not really straight-forward. It is often not possible to 'prove' a custom formula, e.g. for your own erosion mode because not all data is readily accessible. 

-) at least the solver I used was not a 'real' two phase solver. It DID NOT SIMULATE the air in the cavity. 'Air' is just the volume not filled. It is therefore not a 'true two phase solver'. I did many tests regarding the air of my commercial solver, e.g. increasing filling pressure did not influence air bubble size!

Due to all these reasons, it was a goal of mine for years to use an FOSS CFD-package for filling simulations. Little did I know of the problems I would encounter. Today, I think, with the right solver and hardware it's possible to do 90% of what the commercial packages regarding filling (I am still not talking about solidification, I will elaborate later why I don't think it is important to have it in simulation for die-casting and Thixomolding. But it is for any kind of large sand-casting and the like). 

So below I will present findings and results of using a customized solver in OpenFOAM on two filling simulations for Thixomolding (a dummy part and a real part, that might still be in production) and a large sand-casting of a 7 ton submarine propeller. These castings happen on totally different time scales, the THX-castings are 10ms and ~20ms fill time, respectively, and the sand-casting takes about 80s. This has direct consequences to the simulation itself, as we will see later, it is not only the time stepping that is affected. 

Te customized solver used here has the following features:

-) VOF two phase flow in fluid incompressible melt and compressible air) therefore

-) it is multiRegion (fluid and solid)

-) conjugated heat transfer coupling with solid

-) isoAdvection

-) many possibilities for material models as these are standard in OF (for example icoTabulated thermoPhysicalProperties for the liquid metal and perfectGas for the air etc., they can be combined)

-) with a codedFunctionValue at the U inlet boundary condition, the behaviour of the casting machine can be controlled AND regulated. We can introduce pressure limits (also with time retardation etc.), velocity profiles (!), velocity limits that actively react to pressure etc. , basically we can really do what we want. Drawback: some C++ skills required. Advantage: you will learn C++. 

I don't want to leave the drawbacks of this solver unmentioned:

-) it can be highly unstable, only first order ddtSchemes work. I will say this: using commercial packages we do not even know the accuracy of the used time schemes, we are not given this information!

-) it has only worked reliably with simple meshes (castellated) out of snappyHexMesh, but hey, the commercial packages also have this problem?!? What do I mean by 'reliably': a 'fire and forget' type of simulation, that does not need my human interference. It is launched together with necessary DIY control scripts, and it runs until the end. 

-) the meshes in the fluid have to be very uniform, only one cell size allowed

-) we have to dampen U oscillations via limiting functions in controlDict

-) when isoAdvection parameters for alpha.melt (the liquid metal phase) are not perfect the volume of the phase does not remain constant. This applies especially to simulations with many timeSteps. So far, I only found this to be very pronounced in long simulations (sand-casting). Short simulations like for die-casting and Thixomolding do not seem to be affected. However, we can apply a strictVolumeCorrection codedFunction in the controlDict for the alpha.melt liquid phase only if its volume increases. This way, we still have the desired shrinking of the liquid phase with decreasing temperature. Without the correction, the fill time of the sand-casting simulation of the propeller is completely wrong (only a tenth of the calculated value. It almost looks like swelling of polyurethane foam!)! What this correction does is apply the time-dependent integral of the inlet U function to the volume, or in other words: the volume that comes in is in the cavity. The drawaback is: all the currently present liquid volume is divided by a calculated factor, which is not entirely correct: only the volume affected by the isoAdvection should be corrected, that is the boundary interface between liquid and air. Granted, the correction factors are very low PER timeStep, but I should mention this method could well increase the 'bubblyness' of the flow artificially. I don't know if it is possible to correct the volume of the boundary interface only, but there could be an additional problem arising from correcting the volume only at the boundary: what if values < 0 are necessary in one cell (meaning: we need to remove more than is present in a boundary cell. which adjacent cell is the 'correct one', if any, to remove additional alpha.melt to keep the correction uniform at the boundary? Should it even be uniform? Then it really gets tricky very quickly....)

-) initial timeSteps need to be VERY small to get it going

-) maxCo is ramped up via bash script (which you should always do, anyway) to values << 1 in fine steps, which takes a long time. 

-) so far only laminar turbulenceProperties possible, also due to mesh geometry and structure. Astonishingly, this is fulfilled very often in Thixomolding. 

-) using snappyHexMesh we are limited in the cell size of the (less important solid) which causes unnecessarily high cell count in solid (however, not the main driver of computation time). 

-) simulation computationally expensive compared to FD, my estimation is with the ~same mesh size and core count we need 8-10 times more computation time with VOF. In 2025 this is not really an issue because there are workstations with 32+, 64+ and even more cores available. Anyway, with the commercial licenses (and FD!) in a small company you would have 4 or 8-core licenses and a tailored CPU (6, 8 or 12 core for the mentioned licenses, because you'd need slightly more than the 4 or 8 cores for post-processing additionally) to keep CPU frequency high. And this means on this very machine you could not do any other stuff, it would be packed with tasks. Thus, if you arrive at a workstation with 36+ cores for 32 cores in OpenFOAM, your 8-10 fold advantage on commercial FD casting simulation licenses is already gone through the window because the increase with 32 (OpenFOAM) over 4 (commercial casting package) is nearly 8 fold (ok, maybe more like 6-7 fold). Granted this workstation will be a bit more expensive, but not 20000-50000€ per year, the cost of a commercial license casting package. For these expenses, you could easily buy a multi CPU server with hundreds of GB of RAM (don't forget 10 or 25GbE!), or two if refurbished. But, also here, choose your CPUs wisely and leave some spare cores for post-processing (you probably don't want to shove around hundreds of GBs or TBs of data to your your workstations, so you would do PP also on the server). 

## Thixomolding simulation of a dummy part

This part is just a flat plate with automatically generated runners, overflows and vents similar to the geometry presented in this repo: https://github.com/emefff/Geometry-Generation-of-Overflow-and-Venting-Systems-in-Salome-for-Die-Casting-and-Thixomolding 

In a commercial package we would just import the separated .step into the software and go about our business. For use with OpenFOAM we have to:

-) create our groups in the two files (fluid and solid) in Salome. The groups are: inlet, outlet, fluid_to_solid, walls_die. solid_to_fluid will later just be the copy of fluid_to_solid, because it is better to keep it exactly the same. Name them as they should appear in OpenFOAM. 

-) mesh the surface groups in Salome, either with GMSH or Netgen. The most important aspect of this step is to keep nodes of adjacent groups exactly at the same place. For example if your circular inlet has 30 nodes on its boundary with fluid_to_solid, the 30 nodes in fluid_to_solid should also be in exactly the same place. For whatever reason, if you set the meshing parameters the same, this is always the case (I really don't know why, I am guessing because the generatrixes are the same for both overlapping circles). Let me reiterate: Even tiny gaps can and will cause problems in snappyHExMesh. So why even use it?? It is the only FOSS mesher for OpenFOAM that can mesh multiple regions at once.

-) export ASCII .stls of these groups. The file names are the group names. So the inlet becomes inlet.stl and so forth.

-) cp fluid_to_solid.stl solid_to_fluid.stl . We need an exact copy of fluid_to_solid.stl as solid_to_fluid.stl. SnappyHExMesh needs two separate and closed regions, although this seems kind of redundant.

-) still, solid_to_fluid.stl has the wrong names WITHIN the .stl file. So we need to replace all occurrences of "solid fluid_to_solid" and "endsolid fluid_to_solid". Try a "head fluid_to_solid.stl" and a  "tail fluid_to_solid.stl". Execute the following command in the folder: "sed -i 's/fluid_to_solid/solid_to_fluid/g' solid_to_fluid.stl" it will replace all the occurrences of fluid_to_solid. 

-) we need to put the .stls together into two files for our two regions. The fluid region will be called part.stl and the solid region die.stl. Of course, you can call them however you need. So we need to concatenate our separated stls with: "cat inlet.stl outlet.stl fluid_to_solid.stl > part.stl" and "cat walls_die.stl solid_to_fluid.stl > die.stl"

-) copy these two into your constant/triSurface folder

-) use checkSurfaceMesh and surfaceCheck. It is advisable not to ignore presented errors. On the other hand, it is not impossible to get a usable mesh with some errors. I have no clear answer to that. However, if you do not get ANY error, I think you'll be good to proceed in any case.

-) prepare the blockMeshDict anmd snappyHexMeshDict. There are many tutorials for that. First time users definitely need more time. Look for a sufficiently good and uniform castellated ONLY mesh, so deactivate layers and snapped mesh in snappyHExMeshDict. Maybe use a little overhang (larger box) in blockMesh. Start coarse in blockMeshDict and do the rest in refinementRegions and refinementSurfaces in snappyHexMeshDict. Without snapping and layering, meshing this is quite quick. For startes try to keep number of cells in the fluid around 1-2M depending on your machine. In many cases, real-world filling simulations are not better than that with commercial packages. With snappyHexMesh it is difficult to keep the number of cells low in the solid. And because the die always has a larger volume than the fluid, it will definitely need more cells than the fluid. The only option with snappy is to start very coarse with blockMeshDict and refine in snappy, but occasionally you will miss details.

-) If you have a running case with multiRegion, just reuse parts of this. Due to the number of parameters to look at it is very easy to make mistakes. 

-) Determine a maximum velocity (max. inlet velocity times a factor) that will occur in your domain. You should limit the velocity in a limitFields function in the controlDict. Anyway, this will get tricky if it is in the vicinity of the speed of sound. And it will be in die-casting and Thixomolding, because this is the case in real tools. If you have a constriction somewhere in the tool of let's say one fifth of the inlet area and your max. inlet velocity is 50m/s, then you could set your limit to 250m/s. This is not a strict limit, the code introduces dampening factors in the equations that influence the result to approach 250m/s. Only if the simulation crashes, rethink the limit. If the indicated maximum velocity is larger, OF will tell you in the .log. 

























