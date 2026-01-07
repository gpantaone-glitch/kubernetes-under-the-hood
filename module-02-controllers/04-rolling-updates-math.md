# Rolling Updates Are Controlled Failures

Rolling updates feel safe when you first use them. You change an image, apply the Deployment, and Kubernetes gradually replaces old Pods with new ones. Traffic keeps flowing. Nothing crashes visibly. The natural conclusion is that rolling updates are an availability feature.

That conclusion is wrong.

Rolling updates are not designed to maximize availability. They are designed to **bound damage**.

The distinction matters, because misunderstanding it is how outages happen.

When I first used rolling updates, I assumed Kubernetes was trying to keep everything up while upgrading. That assumption frames the process as optimistic: keep services alive and swap bits underneath. What Kubernetes is actually doing is far more conservative.

A rolling update starts from an admission of failure: at some point, Pods *must* be terminated. There is no upgrade path that avoids killing workloads. Kubernetes does not try to hide this fact. It tries to control it.

The core mechanism is simple and brutal. Kubernetes intentionally deletes Pods while strictly limiting how much damage it allows itself to cause at any moment.

This is where the math enters.

Two numbers define the blast radius of a rolling update: how many Pods may be unavailable, and how many extra Pods may temporarily exist. Everything else is derived from these constraints.

If you allow zero unavailable Pods, Kubernetes will create new Pods before deleting old ones. If you allow some unavailability, Kubernetes will delete first and create later. In both cases, Pods are still being killed. The difference is how tightly that killing is constrained.

This reframing explains why rolling updates sometimes feel slow, stubborn, or wasteful. Kubernetes is not optimizing for speed. It is optimizing for **worst-case safety** under uncertainty.

Readiness gates reinforce this conservatism. A Pod is not considered available just because it is running. It must explicitly declare itself ready. Until it does, Kubernetes treats it as dead weight. Progress pauses. Nothing moves forward.

This is not politeness. It is distrust.

Kubernetes assumes new code may be broken, slow, or non-functional. It therefore refuses to count new Pods toward availability until they prove otherwise. Old Pods are kept alive longer than feels necessary because they are the last known good state.

This is also why bad configuration here causes outages.

If availability budgets are miscalculated, Kubernetes will still faithfully enforce them. It will happily delete too many Pods if instructed to do so. The system does not understand your application’s real tolerance for failure. It only understands the numbers you give it.

This leads to an uncomfortable but important realization: rolling updates are not safe by default. They are only as safe as the assumptions encoded in their constraints.

Kubernetes does not know what availability means for your service. It only knows how to enforce a declared budget. If that budget is wrong, the update will fail loudly and correctly.

Once this is understood, the advice to "just increase replicas" starts to sound naive. Replicas do not create safety on their own. Safety comes from bounding how many failures you allow at once.

Rolling updates are therefore best understood not as upgrade mechanisms, but as **failure budgeting systems**. They decide how much of the system may be broken at any moment, and then they intentionally break it within those limits.

That is why rolling updates feel predictable once you stop expecting them to be kind.

They are not trying to keep everything alive.

They are trying to make sure that when things die, they die in controlled numbers.

