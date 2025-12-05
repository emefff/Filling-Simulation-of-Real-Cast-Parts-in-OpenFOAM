# Filling-Simulation-of-Real-Cast-Parts-in-OpenFOAM

Filling simulations of cast parts are very common nowadays. Commercial packages exploiting the finite difference (FD) method exist and are readily available, although at high cost. Historically to my knowledge, all of them are based on the finite difference method due to these simulations being computationally very expensive. From the method alone it directly follows:

- meshes are structured

- mesh geometries are simple, mostly hex meshes are used. 

- Meshes do not follow the part accurately. 'Minecrafty' look of meshes

- implementations of turbulence models are somewhat questionable

- mostly wall functions are used to resolve the boundary layer region

- the package I used projects results onto the geometry. It makes the results look nicer, but it does not improve accuracy. 

Due to them being commercial packages they are:

- closed source

- do they even use turbulence models? (in the case of the package I used, I am not even being informed about this as a user)

- computationally very efficient and user friendly (there's your ONE advantage)

- VERY pricey

- restricted, per-core licenses, which directly leads to

- only running well on low core count CPUs (mostly on workstations, it directly follows that you would buy a dedicated CPU for your 4-core or 8-core license), or if this is not the case, running on servers which are probably not the optimum for this kind of software. 

- restricted regarding exporting the computed data. No, you cannot open any of these files in ParaView or VisIt.

- post-processing is restricted to the features in the package. The formulation of own computed parameters, at least in my used package, was (it probably still is) weird, cumbersome and not really straight-forward. It is often not possible to 'prove' a custom formula, e.g. for your own erosion mode because not all data is readily accessible. 

- at least the solver I used was not a 'real' two phase solver. It DID NOT SIMULATE the air in the cavity. 'Air' is just the volume not filled. It is therefore not a 'true two phase solver'. I did many tests regarding the air of my commercial solver, e.g. increasing filling pressure did not influence air bubble size!

Due to all these reasons, it was a goal of mine for years to use an FOSS CFD-package for filling simulations. Little did I know of the problems I would encounter. Today, I think, with the right solver and hardware it's possible to do 90% of what the commercial packages regarding filling (I am still not talking about solidification, I don't think it is important to have it in simulation for die-casting and Thixomolding. But it is for any kind of large sand-casting and the like). 

So below I will present findings and results of using a customized solver in OpenFOAM on two filling simulations for Thixomolding (a dummy part and a real part, that might still be in production) and a large sand-casting of a 7 ton submarine propeller. These castings take place on totally different time scales, the THX-castings are 10ms and ~20ms fill time, respectively, and the sand-casting takes about 80s. This has direct consequences to the simulation itself, as we will see later, it is not only the time stepping that is affected. For a more realistic casting simulation, it is essential to apply codedFixedValue boundary conditions and also codedFunction in the controlDict. 

The customized solver used here has the following features:

- VOF two phase flow in fluid incompressible melt and compressible air) therefore

- it is multiRegion (fluid and solid)

- conjugated heat transfer coupling with solid

- isoAdvection

- many possibilities for material models as these are standard in OF (for example icoTabulated thermoPhysicalProperties for the liquid metal and perfectGas for the air etc., they can be combined)

- with a codedFixedValue at the U inlet boundary condition, the behaviour of the casting machine can be controlled AND regulated. We can introduce pressure limits (also with time retardation etc.), velocity profiles (!), velocity limits that actively react to pressure etc. , basically we can really do what we want. In the controlDict we can also apply codedFunctions that give more realistic results with this VOF-solver. Drawback: some C++ skills required. Advantage: you will learn C++. 

I don't want to leave the drawbacks of this solver unmentioned:

- it can be highly unstable, only first order ddtSchemes work. I will say this: using commercial packages we do not even know the accuracy of the used time schemes, we are not given this information!

- it has only worked reliably with simple meshes (castellated) out of snappyHexMesh, but hey, the commercial packages also have this problem?!? What do I mean by 'reliably': a 'fire and forget' type of simulation, that does not need my human interference. It is launched together with necessary DIY control scripts, and it runs until the end. 

- the meshes in the fluid have to be very uniform, only one cell size allowed

- we have to dampen U oscillations via limiting functions in controlDict

- when isoAdvection parameters for alpha.melt (the liquid metal phase) are not perfect (they usually aren't) the volume of the phase does not remain constant. This applies especially to simulations with many timeSteps. So far, I only found this to be very pronounced in long simulations (sand-casting). Short simulations like for die-casting and Thixomolding do not seem to be affected tremendously. However, we can apply a strictVolumeCorrection codedFunction in the controlDict for the alpha.melt liquid phase only if its volume increases. This way, we still have the desired shrinking of the liquid phase with decreasing temperature. Without the correction, the fill time of the sand-casting simulation of the propeller is completely wrong (only a tenth of the calculated value. It almost looks like swelling of polyurethane foam!)! What this correction does is apply the time-dependent integral of the inlet U function to the volume, or in other words: the volume that comes in, is also in the cavity (a kind of 'volume conservation'). The drawaback is: all the currently present liquid volume is divided by a calculated factor, which is not entirely correct: only the volume affected by the isoAdvection should be corrected, that is the boundary interface between liquid and air. Granted, the correction factors are very low PER timeStep, but I should mention this method could well increase the 'bubblyness' of the flow artificially. I don't know if it is possible to correct the volume of the boundary interface only, but there could be an additional problem arising from correcting the volume only at the boundary: what if we need to remove more than is present in one single boundary cell? Which adjacent cell is the 'correct one', if any, to remove additional alpha.melt to keep the correction uniform at the boundary? Should it even be uniform? Then it really gets tricky very quickly....

- initial timeSteps need to be VERY small to get it going

- maxCo is ramped up via bash script (which you should always do, anyway) to values << 1 in fine steps, which takes a long time. 

- so far only laminar turbulenceProperties possible, also due to mesh geometry and structure. To the laymen it will be a surprise, but this is indeed fulfilled very often in Thixomolding (hard to believe with Uinlet > 40m/s). 

- using snappyHexMesh we are limited in the cell size of the (less important) solid which causes unnecessarily high cell count in solid (however, not the main driver of computation time). 

- simulation computationally expensive compared to FD, my estimation is with the ~same mesh size and core count we need 8-10 times more computation time with VOF. In 2025 this is not really an issue because there are workstations with 32+, 64+ and even more cores available. Anyway, with the commercial licenses (and FD!) in a small company you would have 4 or 8-core licenses and a tailored CPU (6, 8 or 12 core for the mentioned licenses, because you'd need slightly more than the 4 or 8 cores for post-processing additionally) to keep CPU frequency high. And this means on this very machine you could not do any other stuff, it would be packed with tasks. Thus, if you arrive at a workstation with 36+ cores for 32 cores in OpenFOAM, your 8-10 fold advantage on commercial FD casting simulation licenses is already gone through the window because the increase with 32 (OpenFOAM) over 4 (commercial casting package) is nearly 8 fold (ok, maybe more like 6-7 fold). Granted this workstation will be a bit more expensive, but not 20000-50000€ per year, the cost of a commercial license casting package. For these expenses, you could easily buy a multi CPU server with hundreds of GB of RAM (don't forget 10 or 25GbE!), or two if refurbished. But, also here, choose your CPUs wisely and leave some spare cores for post-processing (you probably don't want to shove around hundreds of GBs or TBs of data to your workstations, so you would do PP also on the server). In a commercial package we would just import the separated .step into the software and go about our business. For use with OpenFOAM we have to:

- create our surface groups in the two files (fluid and solid) in Salome. The groups are: inlet, outlet, fluid_to_solid, walls_die. solid_to_fluid will later just be the copy of fluid_to_solid, because it is better to keep it exactly the same. Name them as they should appear in OpenFOAM. 

- mesh the surface groups in Salome, either with GMSH or Netgen. The most important aspect of this step is to keep nodes of adjacent groups exactly in the same place. For example if your circular inlet has 30 nodes on its boundary with fluid_to_solid, the same 30 nodes in fluid_to_solid should also be in exactly the same place. Because the generatrix of the circle (or whatever geometry) is the same, setting the same meshing parameters is sufficient to get the same nodes in both surface meshes. Let me reiterate: Even tiny gaps can and will cause problems in snappyHexMesh. So why even use it?? It is the only FOSS mesher for OpenFOAM that can mesh multiple regions at once. 

- export ASCII .stls of these groups. The file names are the group names. So the inlet becomes inlet.stl and so forth.

- "cp fluid_to_solid.stl solid_to_fluid.stl" We need an exact copy of fluid_to_solid.stl as solid_to_fluid.stl. SnappyHexMesh needs two separate and closed regions, although this seems kind of redundant.

- still, solid_to_fluid.stl has the wrong names WITHIN the .stl file. So we need to replace all occurrences of "solid fluid_to_solid" and "endsolid fluid_to_solid". Try a "head fluid_to_solid.stl" and a  "tail fluid_to_solid.stl". Execute the following command in the folder: "sed -i 's/fluid_to_solid/solid_to_fluid/g' solid_to_fluid.stl" it will replace all the occurrences of fluid_to_solid. 

- we need to put the .stls together into two files for our two regions. The fluid region will be called part.stl and the solid region die.stl. Of course, you can call them however you need. So we need to concatenate our separated stls with: "cat inlet.stl outlet.stl fluid_to_solid.stl > part.stl" and "cat walls_die.stl solid_to_fluid.stl > die.stl"

- copy these two into your constant/triSurface folder

- use checkSurfaceMesh and surfaceCheck. It is advisable not to ignore presented errors. On the other hand, it is not impossible to get a usable mesh with some errors. I have no clear answer to that. However, if you do not get ANY error, I think you'll be good to proceed in any case.

- prepare the blockMeshDict and snappyHexMeshDict. There are many tutorials for that. First time users definitely need more time. Look for a sufficiently good and uniform castellated ONLY mesh, so deactivate layers and snapped mesh in snappyHexMeshDict. Maybe use a little overhang (larger box) in blockMesh. Choose good locationInMesh for both regions. Start coarse in blockMeshDict and do the rest in refinementRegions and refinementSurfaces in snappyHexMeshDict. Without snapping and layering, meshing this is quite quick. For starters try to keep number of cells in the fluid around 1-2M depending on your machine. In many cases, real-world filling simulations are not better than that with commercial packages. With snappyHexMesh it is difficult to keep the number of cells low in the solid. And because the die always has a larger volume than the fluid, it will definitely need more cells than the fluid. The only option with snappy is to start very coarse with blockMeshDict and refine in snappy, but occasionally you will miss details. I have thought about reducing the die to a 'wrap' around the fluid, but this is quite tricky to do with complex tools. 

- determine a maximum velocity that will occur in your domain (max. inlet velocity times a factor, minimum of cross-section in flow divided by inlet area). You should limit the velocity in a limitFields function in the controlDict. Anyway, this will get tricky if the maximum is near the speed of sound. And it will be in die-casting and Thixomolding, because this is the case in real tools (the designer must make sure the speed of sound is NEVER surpassed in the tool. It will constrict the flow, otherwise). If you have a constriction somewhere in the tool of, let's say one fifth of the inlet area and your max. inlet velocity is 50m/s, then you could set your limit to 250m/s. This is not a strict limit, the code introduces dampening factors in the equations that influence the result to approach 250m/s. Only if the simulation crashes for that reason, rethink the limit (it could also be lower than your real limit, but not too low...). If the indicated maximum velocity is larger, OF will tell you in the .log. You can also change the limit dynamically in a bash script, like is done here (inlet velocity times factor, if you ramp up the speed in the beginning of the shot, the limit will also follow. Very convenient.)

- Define U curve for the inlet. Do you need to stop the simulation automatically at a certain filling percentage? Do you need to ramp down U before the end of the simulation at a certain filling percentage? Both can be done with a bash script or with a codedFunction in the controlDict. Do you need to ramp down the U field in the inlet when the pressure surpasses the limits of your casting machine (or: more realistically with a time delay, when the high pressure peak passed through your mnachinery at the speed of sound in your materials until it reached the inlet)? CodedFunctions can do this. You should look into them, they give ultimate freedom without fiddling around with the solver. Some C++ skills are needed though.

## Thixomolding simulation of a dummy part

This part is just a flat plate with automatically generated runners, overflows and vents similar to the geometry presented in this repo: https://github.com/emefff/Geometry-Generation-of-Overflow-and-Venting-Systems-in-Salome-for-Die-Casting-and-Thixomolding 

This was the first part I simulated with the customized solver, the goal was to find out if it even works and what has to be done to get a stable, 'bullet-proof' simulation with this sensitive solver. In the end, the results were quite convincing. 

The codedFixedValue in the U inlet lets us ramp up the speed of the screw, just like in the real machine. We do not have a frozen plug to blow out, so we do not get the high pressure peak in the beginning. I guess this could be implemented artificially, but it is of minor interest. The U curve at the inlet is presented in the following image. It ramps up from 4.5 to 45m/s in 5ms. Velocity is reduced by 1% in a timeStep when pressureMax of 1000bar is surpassed (pressureLimiter), if pressure is below 1000bar the already applied velocity is kept. When a phaseFraction of 98% is reached, velocity U is ramped down. At 99% the simulation is stopped this is a common value. 

<img width="1602" height="1556" alt="Bildschirmfoto vom 2025-11-17 13-16-16" src="https://github.com/user-attachments/assets/117f6b27-9995-47d0-a9d0-37b69f6677ed" />

As mentioned, inlet velocity ramps up till 5ms to 45m/s. We could also do this part of the curve with a standard inlet BC with tabulated values. The following part in the curve cannot be done with any of the standard BCs. Until 7.5ms or so we find the velocity remains constant, then it drops to 32m/s and at 8.5ms it sits around 24m/s. Between these two, the pressure limiter kicked in and reduced the inlet velocity in a regulated fashion in some of the timeSteps in between. The pressure curve looks like this:

<img width="1602" height="1554" alt="Bildschirmfoto vom 2025-11-17 13-18-17" src="https://github.com/user-attachments/assets/52551d74-5bc3-4dd3-a03d-a201e0d46051" />

Disregarding the non-existing pressure peak by the non-existing cold plug in this simulation, the pressure curve looks quite typical (not a lot of pressure during filling, but a lot in the end). Only very late in the filling, pressure increases to high values. It is the time when outlets are reached by the alpha.melt phase, their cross-sections are usually smaller so it is quite natural we get a higher pressure. Further increases of pressure are common when vents get filled by the metal. Usually, they feature even lower cross-sections than the outlet connecting to the overflows (see image below for reference).

The affected timeSteps are documented in the log file. In the end around 9.5ms, velocity drops to very values because the phaseFraction reached 99%. Beforehand, we do not know exactly WHEN in the simulation 99% is reached. It is very likely we do not hit this point in time, or just before, with a saved result. It is therefore also convenient to decrease writeInterval in the controlDict in an automated fashion. The shared bash script limitULimiter does a few things: it sets the U limit in the controlDict to a prescribed value (it follows the inlet velocity), it decreases the writeInterval at 98%, it kills all processes running this solver at 99%. 

Let's look at an image of the velocity in the liquid metal at 5ms, when the maximum of set inlet velocity is reached for the first time:

![U_1](https://github.com/user-attachments/assets/072c8e79-3076-476e-b0e4-64146ece7df0)

Granted, the mesh doesn't look perfect especially at the overflows. But what if I told you people are paying 50000€ per license and year + personnel costs for similar results?
A fancy video of this simulation can also be found on YT: https://www.youtube.com/watch?v=danuRmXGdOo
The video starts quite unusual by showing both phases, air and liquid metal. This is somewhat unusual, but in the in the end, only the metal phase is shown also. Why show it like this? Because commercial casting solvers CANNOT DO this, they neglect simulation of the air. The casting engineer also needs the results of the air, he needs to know if and where the speed of sound is reached (he needs to apply corrections if this is the case!). Even today, this is mostly evaluated by hand calculations!

The second simulation  was done on the geometry of real production part. I will and cannot name the customer, but this part is part of an E-Bike assembly widely used in Europe. Therefore, I also have to apply some blurring as not to give away who made this part. 

## Thixomolding simulation of a real production part

Now what is this about? This step is necessary for me to check if the customized solver is off in some parameter or not. The first and most important check is to verify if the fill time is correct. This is when I first suspected that there had to be some corrections made when using this VOF solver. But back then I was not aware how bad uncorrected results would be with sand-casting. For some reason here the effect of a temperature dependent density were when applied with the rho0, beta formula were strange, the fill time was off by a few ms. Quite a lot when the calculated fill time at a constant 45m/s is only 7.5ms! However, it went away with using icoTabulated values in thermoPhysicalProperties for equationOfState (rho), thermodynamics (Hs, Cp), transport (mu, kappa). That's all. 

In this simulation I tried if the codedFixedValue can also be pressure regulated at the very end of the filling. That is, U is regulated in such a manner like the packing pressure does on a Thixomolding or die-casting machine (phase 1 is velocity controlled, phase 2 of filling is usually pressure controlled). Nevertheless, U can never be larger than the set maximum velocity. 

The velocity profile of the real part looks like this:

![UInlet_vs_time](https://github.com/user-attachments/assets/fc161717-df84-4e92-ab21-807248ba0e72)

Although the diagram reaches 14ms, the 98% limit is reached at ~10.09ms which is in reasonable agreement with the 7.5ms fill time at 45m/s considering the ramp in the beginning. So the end of this diagram is not really important because the pressure regulation just takes a long time. This is also always the question on a real machine, phase 2 can never account fully for the fill time.
The pressure curve looks interesting because the limiter does not always seem to work, there are some overshoots above 1000bar:

![pInlet_vs_time](https://github.com/user-attachments/assets/1affa780-6b57-41d8-a6ef-ed6656f5545d)

The reason remains unclear, it could be just because with the codedFixedValue we can only react to a pressure that is already over the limit and if the timeStep is large (this is what we want, because we cannot wait forever to complete the simulation) large overshoots can happen. so with smaller timeStepping we should be able to get a flatter curve. In this case, there is no clear separation of phase 1 and phase 2 in the shot, like on a real machine. If you look at the code, the user cannot foresee when the speed drops in a timeStep and is then increased to match the pressure of 1000bar. But separating the two phases is clearly possible when we, for example, increase the U inlet such that 1000bar are reached only after a certain percentage of filling. This should be easy.

The results are within agreement of the commercial software, like mentioned before, here's a blurred image of the real cast part at 6ms when velocity is still high:

<img width="2908" height="1400" alt="Bildschirmfoto vom 2025-11-17 14-31-26" src="https://github.com/user-attachments/assets/976b5060-a38f-4e01-a654-8a46de0d3e41" />

Obviously, we reach high velocities just before the part and also after the inlets (the inlets at the part not the inlet in the simulation). Clearly, concerns of increased erosion in these areas were mentionend. But there's no way out: the part doesn't allow for larger inlet and runner and, if I remember correctly, there was no desire to build a two cavity tool.

The simulation of this part and others will have to be repeated with the findings from the following simulation which is of a submarine propeller sand casting. 

## Sand-casting simulation of a submarine propeller

The submarine propeller used here is a fictitious propeller designed by a talented engineer on GrabCAD (today it's not available anymore, for unknown reasons). The geometry was already used in the repository showing results of a flow simulation of the very same propeller with diffuser and a fictitious submarine: https://github.com/emefff/Thrust-And-Flow-Of-Submarine-Propeller-In-OpenFOAM
Scaled up to approx. 5m in diameter the result is a volume of 0.863m³ total casting volume with risers, basin, runners and vents. I have to admit here, this is the first design of a sand casting for me, so there might be some errors in the setup. However, runner diameters and pressure calculation were done following a casting standard for these types of castings. After all, it is purely a test case for the solver. The designed total casting is placed into a sand mould, in reality it would of course not entirely be sand, but after all only the material in contact with the liquid metal is important. From literature I have learned, that propeller alloys are called 'bronzes' although they are really more like brasses. So the material parameters here used are very brass-like. This is the model used for the simulation, a life-size person (https://grabcad.com/library/bossy-girl-1) was inserted for comparison:

<img width="2874" height="1656" alt="Bildschirmfoto vom 2025-11-17 15-00-52" src="https://github.com/user-attachments/assets/42987c6e-57d4-49c4-a815-627f172e49c2" />

With this simulation, I encountered the following problem, which was already mentioned above briefly: the volume of the liquid phase present in the cavity increased with every timeStep. In the CFD community this is known as "non-physical phase gain" or "numerical volume increase". However, I was not aware of this until I made this simulation with this customized solver. In this paper, this phenomenon is discussed also regarding an OpenFOAM solver: https://pmc.ncbi.nlm.nih.gov/articles/PMC9145428/ 

There you can find the following paragraph: 
> "MULES is the default interface description method in OpenFOAM phase change solvers and has been used in extensive phase change problems. The VOF class has a non-sharp character, meaning it produces a smeared interface between phases, resulting in the non-accurate calculation of interfacial properties and generating spurious currents. The existence of non-physical spurious currents leads to the increased interfacial mass transfer while simulating condensation and evaporation. The mentioned scenario contributes to high numerical errors in such simulations and can be encountered as the chief shortcoming of VOF."

Now, although this customized solver does not use any phaseChange, it still has to compute the boundary phase between liquid metal and air. And in my previous runs it was obvious that the volume is not correct. 

However, to counteract this numerical phenomenon, a strictVolumeCorrection in the form of a codedFunction in the controlDict has been applied. Because the inflow at the inlet and the already present phase in the basin was determined by the user, we apply some simple calculus to get the volume fraction at any time (we need to perform an integration over time on the inlet U curve to get the time-dependent volumeFraction within the cavity). Quite unnecessary in retrospect, I did a ramp of the U inlet until t=1e-3s. Both files, the U BCs and the controlDict are shared, so you will understand when looking at them. A typical log file entry by the codedFunction would look like this:

> #####Volume correction: Time = 47.41622s, Volume correction factor = 0.9999869, actualVolFrac = 0.5622349, inletVolFrac = 0.5622276

The correction factor is usually very small, but due to the amount of iterations the difference between corrected and uncorrected volume is very substantial. 

This simulation is still running as of today (17/11/2025), so the results presented below are from the casting approx. half-filled: 

![U_1](https://github.com/user-attachments/assets/b0a667e6-1d15-47de-8eb2-2bffd5814e03)

The predicted velocities are in agreement with the calculations done before. However, the level in the basin is now quite high, the choke point in the runner does not seem to get along with the defined flow. Due to the long simulation time, this time we see a real influence on T.melt. However, this time and also due to the numerical phenomenon encountered, the material parameters do not include any increase of viscosity during cooling (there will be no freezing of metal visible in this simulation):

![Tmelt_1](https://github.com/user-attachments/assets/1003d0a8-b64e-4805-86d7-ab326c9f4cf5)

The final results will still take some time, at least one more week. 

UPDATE 12/05/2025: I finally gave up on this. The estimated fill time was ~80s for this ~7 ton casting. At around 54s weird things start to happen: U overshoots and the strictVolumeCorrection does not seem to be effective any more. Therefore, after simulating several weeks and considerable electricity cost (consumption amounts to 16kWh per day on this workstation) my verdict is: this method may be not usable for slow casting processes (all gravity casting, sand-casting etc.) at all. Perhaps the strictVolumeCorrection should be applied to the boundary zone only.

emefff@gmx.at








