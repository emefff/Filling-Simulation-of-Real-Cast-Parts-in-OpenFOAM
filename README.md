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

Due to all these reasons, it was a goal of mine for years to use an FOSS CFD-package for filling simulations. Little did I know of the problems I would encounter. Today, I think, with the right solver and hardware it's possible to do 90% of what the commercial packages regarding filling (I am still not talking about solidification, I will elaborate later why I don't think it is important to have it in simulation for die-casting and Thixomolding. But it is for any kind of large sand-casting and the like). 

So below I will present findings and results of using a customized solver in OpenFOAM on two filling simulations for Thixomolding (a dummy part and a real part, that might still be in production) and a large sand-casting of a 7 ton submarine propeller. These castings happen on totally different time scales, the THX-castings are 10ms and ~20ms fill time, respectively, and the sand-casting takes about 80s. This has direct consequences to the simulation itself, as we will see later it is not only the time stepping that is affected. 

Te customized solver used here has the following features:

-) VOF two phase flow in fluid incompressible melt and compressible air)

-) it is multiRegion (fluid and solid)

-) conjugated heat transfer coupling with solid

-) isoAdvection

-) many possibilities for material models as these are standard in OF (for example icoTabulated thermoPhysicalProperties for the liquid metal and perfectGas for the air etc., they can be combined)

I don't want to leave the drawbacks of this solver unmentioned:

-) it has only worked reliably with simple meshes (castellated) out of snappyHexMesh, but hey, the commercial packages also have this problem?!?

-) the meshes in the fluid have to be very uniform, only one cell size allowed

-) we have to dampen U oscillations via limiting functions in controlDict

-) initial timeSteps need to be VERY small to get it going

-) maxCo is ramped up via bash script (which you should always do, anyway) 

-) so far only laminar turbulenceProperties possible, also due to mesh geometry and structure. Astonishingly, this is fulfilled very often in Thixomolding. 

-) simulation computationally expensive compared to FD, my guess is with the same core count we need 8-10 times more computation time. In 2025 this is not really an issue because there are workstations with 32, 64 and even more cores available. Anyway, with the commercial licenses (and finite difference!) in a small company you would have 4 or 8-core licenses and a tailored CPU (6, 8 or 12 core for the mentioned licenses, because you'd need slightly more than the 4 or 8 cores for postprocessing additionally) to keep CPU freq high. And that means on this very machine you could not do any other stuff, it would be packed with tasks. Thus, if you arrive at a workstation with 36 cores for 32 cores in OpenFOAM, your 8-10 fold advantage on commercial FD casting simulation licenses is already gone through the window because the increase with 32 (OpenFOAM) over 4 (commercial casting package) is nearly 8 fold. Granted this workstation will be a bit more expensive, but not 20000-50000€ per year, the price of a commercial license casting package. 




















