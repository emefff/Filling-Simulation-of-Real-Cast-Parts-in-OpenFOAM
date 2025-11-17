# Filling-Simulation-of-Real-Cast-Parts-in-OpenFOAM

Filling simulations of cast parts are very common nowadays. Commercial packages exploiting the finite difference method exist and are readily available, although at a premium price. Historically, to my knowledge, all of them are based on the finite difference method due to these simulations being computationally very expensive. From the method alone it directly follows:

-) meshes are structured

-) mesh geometries are simple, mostly hex meshes are used

-) implementations of turbulence models are somewhat questionable

-) mostly wall functions are used to resolve the boundary layer region


Due to them being commercial packages they are:

-) closed source

-) do they even use turbulence models? (in the case of the package I used, I am not even being informed about this as a user)

-) computationally very efficient and user friendly (there's your ONE advantage)

-) VERY expensive

-) restricted, per-core licenses, which directly leads to

-) running on low core count CPUs (mostly on workstations, it directly follows that you would by it especially buy it for your 4-core or 8-core licnese), or if it this is not the case, running on servers which are probably not optimized for this kind of software. 

-) restricted regarding exporting the computed data. No, you cannot open any of these files in ParaView or VisIt.

Due to all these reasons, it was a goal of mine for years to use an FOSS CFD-package for filling simulations. Little did I know of the problems I would encounter. Today, I think, with the right solver and hardware it's possible to do 90% of what the commercial packages regarding filling (I am still not talking about solidification, I will elaborate later why I don't think it is important to have it in simulation for die-casting and Thixomolding. But it is for any kind of large sand-casting and the like). 

So below I will present findings and results of using a customized solver in OpenFOAM on two filling simulations for Thixomolding (a dummy part and a real part, that might still be in production) and a large sand-casting of a 7 ton submarine propeller. These castings happen on totally different time scales (the THX-castings are 10ms and ~20ms fill time respectively and the sand-casting takes about 80s) with direct consequences to the simulation itself, as we will see later it is not only the time stepping that is affected. 










