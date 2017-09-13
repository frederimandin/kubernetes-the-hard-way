# Cleaning Up

In this labs you will delete the compute resources created during this tutorial.

As everything has been setup in the same resource group, we only have to delete that group, although it may take a while:

```
azure group delete -n $GROUP
```
