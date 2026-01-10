ServiceAccounts Are Identities (Traced End-to-End)
A Pod does not wake up with permissions.
A Pod does not authenticate itself.
A Pod does not “log in” to Kubernetes.
Yet Pods routinely talk to the API server and get real work done.
So the only honest question is:
Where does that identity actually come from, and where is it enforced?
Anything less precise is hand-waving.
*******
