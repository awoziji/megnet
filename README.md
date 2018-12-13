# MatErials Graph Networks (MEGNet) for molecule/crystal property prediction

## Theory

Graph networks[1] are a superclass of graph-based neural networks. There are a few innovations compared to conventional graph-based neural neworks. 

* Global state attributes are added to the node/edge graph representation. These features work as a portal for structure-independent features such as temperature, pressure etc and also are an information exchange placeholder that facilitates information passing across longer spatial domains. 
* The update function involves the message interchange among all three levels of information, i.e., the node, bond and state information. Thus it is one of the most general model. 

![](./resources/model_diagram.png)
<div align='center'><strong>Figure 1. The graph network update function.</strong></div>

The `MEGNet` model implements two major components: one is the `graph network` layer and the other is the `set2set` layer. The layers are based on `keras` API and is thus compatible with other keras modules. 

Different crystals/molecules have different number of atoms. Therefore it is impossible to use data batches without padding the structures to make them uniform in atom number. 

`MEGNet` Takes another approach. Instead of make structure batches, we assemble many structures into one 'gaint' structure and this structure has a vector output with each entry being the target value for the corresponding structure. Therefore, the batch number is always `1`. 

Assuming a structure has N atoms and M bonds, a structure graph is represented as **V**, **E** and **u**, where **V** is a N\*Nv matrix, **E** comprises of three parts, one for the bond attributes, a M\*Nm matrix and index pairs (rk, sk) for the bonds, and **u** is a vector with length Nu. We vectorize rk and sk to form `index1` and `index2`, both are vectors with length Nm. In summary, the graph is a data structure with **V** (N\*Nv), **E** (M\*Nm), **u** (Nu, ), `index1` (M, ) and `index2` (M, ). 

We then assemble several structures together. For **V**, we directly append the atomic attributes from all structures, forming a matrix (1\*N'\*Nv), where N' > N. To indicate the belongingness of each atom attribute vector, we use a `atom_ind` vector. For example if `N'=5` and the first 3 atoms belongs to the first structure and the remainings the second structure, our `atom_ind` vector would be `[0, 0, 0, 1, 1]`. For the bond attribute, we perform the same appending method, and use `bond_ind` vector to indicate the bond belongingness. For `index1` and `index2`, we need to shift the integer values. For example, if `index1` and `index2` are `[0, 0, 1, 1]` and `[1, 1, 0, 0]` for structure 1 and are `[0, 0, 1, 1]` and `[1, 1, 0, 0]` for structure two. The assembled indices are `[0, 0, 1, 1, 2, 2, 3, 3]` and `[1, 1, 0, 0, 3, 3, 2, 2]`. Finally **u** expands a new dimension to take into account of the number of structures, and becomes a `1\*Ng\*Nu` tensor, where Ng is the number of graphs. `1` is added as the first dimension of all inputs because we fixed the batch size to be 1 (1 `gaint` graph) to comply the keras inputs requirements. 

In summary the inputs for the model is **V** (1\*N'\*Nv), **E** (1\*M'\*Nm), **u** (1\*Ng\*Nu), `index1` (1\*M'), `index2` (1\*M'), `atom_ind` (1\*N'), and `bond_ind` (1\*M'). For Z-only atomic features, **V** is a (1\*N') vector. 
 
## Usage

A fast model building tool is in the `megnet.model` submodule, and the corresponding tests explain the usage mechanisms. 

A simple model is as follows. 

```python
from keras.layers import Input, Dense
from keras.models import Model
from megnet.layers import MEGNet, Set2Set

n_atom_feature= 20
n_bond_feature = 10
n_global_feature = 2

# Define model inputs
int32 = 'int32'
x1 = Input(shape=(None, n_atom_feature)) # atom feature placeholder
x2 = Input(shape=(None, n_bond_feature)) # bond feature placeholder
x3 = Input(shape=(None, n_global_feature)) # global feature placeholder
x4 = Input(shape=(None,), dtype=int32) # bond index1 placeholder
x5 = Input(shape=(None,), dtype=int32) # bond index2 placeholder
x6 = Input(shape=(None,), dtype=int32) # atom_ind placeholder
x7 = Input(shape=(None,), dtype=int32) # bond_ind placeholder
xs = [x1, x2, x3, x4, x5, x6, x7]

# Pass the inputs to the MEGNet layer
# Here the list are the hidden units + the output unit, 
# you can have others like [n1] or [n1, n2, n3 ...] if you want. 
out = MEGNet([32, 16], [32, 16], [32, 16], pool_method='mean', 
activation='relu')(xs)

# the output is a tuple of new graphs V, E and u
# Since u is a per-structure quantity, 
# we can directly use it to predict per-structure property
out = Dense(1)(out[2])

# Set up the model and compile it!
model = Model(inputs=xs, outputs=out)
model.compile(loss='mse', optimizer='adam')
```
With less than 20 lines of code, you have built a graph network model that is ready for all material property prediction!

## References

[1] Battaglia, Peter W., et al. "Relational inductive biases, deep learning, and graph networks." arXiv preprint arXiv:1806.01261 (2018).