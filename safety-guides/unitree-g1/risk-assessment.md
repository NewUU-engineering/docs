# Unitree G1 Risk Assessment

The main hazards of working with the lab's Unitree G1, what could happen, and how we prevent it. Most hazards are unlikely but severe, so every high-severity row needs all its measures in place before power-on. The ratings are first estimates for the supervisor to confirm. Section numbers refer to the [safety guide](README.md), and page numbers to the [Unitree G1 User Manual V1.1](https://reliablerobotics.ai/wp-content/uploads/2025/03/G1-User-Manual_compressed.pdf).

| Hazard | What could happen | Likelihood | Severity | Preventive measures |
|---|---|---|---|---|
| Robot falls during operation | Injury to nearby feet and legs; damage to the robot | Medium | High | Frame for starts, shutdowns and anything new; 2 m distance; flat floor; low speed; stop tested every session (sections 4, 5, 7) |
| Wrong button pressed, e.g. zero torque (L2 + Y) on a standing robot | Robot drops at once | Medium | High | First-time users learn the label before their first session; only L2 + B is used to stop; new modes only with approval (section 3) |
| Robot let go of in locked standing | It doesn't balance, so it tips over | Medium | High | Keep the rope hooked or a person holding it until running or regular mode is on (sections 3, 5) |
| Fingers caught in a moving joint | Crushed or cut fingers | Medium | High | Never touch a moving robot; hold only the back of the shoulders in a seated start; hands off the joints when lifting (sections 5, 6, 8) |
| Robot walks into a person or object | Bruises; damaged equipment | Medium | Medium | No walking in our crowded lab, only in an approved larger space; 2 m distance; spotter; low speed (sections 4, 6) |
| Unexpected movement from your own program | Injury; robot damage | Medium | High | Simulation and a second simulator first; code review; frame stage; joint-limit clipping; tested stop (section 9) |
| Your code conflicts with the built-in controller | Jitter or loss of balance (p.3, p.21) | Medium | High | Develop mode for low-level code; only one controller in charge in high-level code (section 9) |
| Remote dies or disconnects | The robot can't be stopped from the remote | Low | High | Remote charged before every session; DL light checked; forced-shutdown steps known (sections 7, checklist) |
| Battery runs flat during operation | Robot falls (p.4) | Medium | High | Battery charged before the session; stop when the last LED flashes; long breaks hanging in damping (sections 5, 7) |
| Shutdown while standing on its own | Robot falls flat (p.25) | Low | High | Shut down only while hanging or seated (section 5) |
| Battery overheats or catches fire while charging | Fire, burns, toxic smoke | Low | High | Charged out of the robot, switched off and cooled; voltage matches; never unattended; powder extinguishers just outside the lab (sections 4, 8) |
| Battery damaged after a fall or impact | Delayed fire or swelling | Low | High | After-a-fall battery check; damaged packs labelled and set aside (section 8, checklist) |
| Lifting or carrying the 35 kg robot | Back injury; pinched hands; robot dropped | Medium | Medium | Two-person lift, hands off the joints; packing from the frame (section 8) |
| Rope or frame fails | Robot drops | Low | High | Frame installed correctly with wheels locked; rope, buckles and knot checked every session (section 4, checklist) |
| Tripping over cables or rope | Someone falls | Medium | Low | Cables taped outside the zone; rope coiled when not in use (section 4) |
| Someone else connects over the network | Unauthorised control of the robot | Low | High | Default passwords changed; separate robot network (section 9) |
