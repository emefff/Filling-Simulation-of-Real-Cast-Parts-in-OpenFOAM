# Filling-Simulation-of-Real-Cast-Parts-in-OpenFOAM

Filling simulations of cast parts are very common nowadays. Commercial packages exploiting the finite difference (FD) method exist and are readily available, although at a premium price. Historically, to my knowledge, all of them are based on the finite difference method due to these simulations being computationally very expensive. From the method alone it directly follows:

-) meshes are structured

-) mesh geometries are simple, mostly hex meshes are used

-) implementations of turbulence models are somewhat questionable

-) mostly wall functions are used to resolve the boundary layer region


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

-) initial timeSteps need to be VERY small to get it going

-) maxCo is ramped up via bash script (which you should always do, anyway) 

-) so far only laminar turbulenceProperties possible, also due to mesh geometry and structure. Astonishingly, this is fulfilled very often in Thixomolding. 

-) using snappyHexMesh we are limited in the cell size of the (less important solid) which causes unnecessarily high cell count in solid (however, not the main driver of computation time). 

-) simulation computationally expensive compared to FD, my estimation is with the same core count we need 8-10 times more computation time with VOF (~same mesh size in fluid). In 2025 this is not really an issue because there are workstations with 32+, 64+ and even more cores available. Anyway, with the commercial licenses (and FD!) in a small company you would have 4 or 8-core licenses and a tailored CPU (6, 8 or 12 core for the mentioned licenses, because you'd need slightly more than the 4 or 8 cores for post-processing additionally) to keep CPU frequency high. And that means on this very machine you could not do any other stuff, it would be packed with tasks. Thus, if you arrive at a workstation with 36+ cores for 32 cores in OpenFOAM, your 8-10 fold advantage on commercial FD casting simulation licenses is already gone through the window because the increase with 32 (OpenFOAM) over 4 (commercial casting package) is nearly 8 fold. Granted this workstation will be a bit more expensive, but not 20000-50000€ per year, the price of a commercial license casting package. For these expenses, you could easily buy a multi CPU server with hundreds of GB of RAM (don't forget 10 or 25GbE!), or two if refurbished. But, also here, choose your CPUs wisely and leave some spare cores for post-processing (you probably don't want to shove around hundreds of GBs or TBs of data to your your workstations, so you would do PP also on the server). 























