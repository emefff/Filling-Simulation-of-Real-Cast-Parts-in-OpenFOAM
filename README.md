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


