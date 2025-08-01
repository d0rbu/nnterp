Adding Support for Your Model
==============================

``nnterp`` uses a standardized naming convention to provide a unified interface across transformer architectures. When your model doesn't follow the expected naming patterns, you can use ``RenameConfig`` to map your model's modules to the standardized names.

Understanding the Target Structure
----------------------------------

``nnterp`` expects models to follow this structure:

.. code-block:: text

   StandardizedTransformer
   ├── model.layers[i]
   │   ├── self_attn
   │   └── mlp
   ├── ln_final
   └── lm_head

All models are automatically renamed to match this pattern using built-in mappings for common architectures.

Basic RenameConfig Usage
------------------------

When automatic renaming fails, create a custom ``RenameConfig``:

.. code-block:: python

   from nnterp import StandardizedTransformer
   from nnterp.rename_utils import RenameConfig

   # Hypothetical model with custom naming
   rename_config = RenameConfig(
       model_name="custom_transformer",           # Maps to "model"
       layers_name="custom_layers",               # Maps to "layers"
       attn_name="custom_attention",              # Maps to "self_attn"
       mlp_name="custom_ffn",                     # Maps to "mlp"
       ln_final_name="custom_norm",               # Maps to "ln_final"
       lm_head_name="custom_head",                # Maps to "lm_head"
       mlp_returns_tuple=True                     # Some models, mostly MoE, have mlps that return tuples where the first element is the d_model output of the MLP
   )

   model = StandardizedTransformer(
       "your-model-name",
       rename_config=rename_config
   )

Path-Based Renaming
-------------------

For nested modules, use dot notation to specify the full path:

.. code-block:: python

   rename_config = RenameConfig(
       layers_name=".custom_transformer.encoder_layers",
       ln_final_name=".custom_transformer.final_norm"
   )

Multiple Alternative Names
--------------------------

Provide multiple options for the same component:

.. code-block:: python

   rename_config = RenameConfig(
       attn_name=["attention", "self_attention", "mha"],
       mlp_name=["ffn", "feed_forward", "mlp_block"]
   )

Real Example: GPT-J Support
----------------------------

Here's how GPT-J attention probabilities were added to nnterp:

First, examine the model architecture:

.. code-block:: python

   from nnterp import StandardizedTransformer
   
   # GPT-J loads with basic renaming but attention probabilities fail
   model = StandardizedTransformer("yujiepan/gptj-tiny-random")
   # Warning: Attention probabilities test failed

Locate the attention probabilities in the forward pass:

.. code-block:: python

   # Find where attention weights are computed
   with model.scan("test"):
       model.attentions[0].source.self__attn_0.source.self_attn_dropout_0.output.shape
       # Shape: (batch, heads, seq_len, seq_len) - this is what we want

Create the attention probabilities function:

.. code-block:: python

   from nnterp.rename_utils import AttnProbFunction, RenameConfig

   class GPTJAttnProbFunction(AttnProbFunction):
       def get_attention_prob_source(self, attention_module, return_module_source=False):
           if return_module_source:
               return attention_module.source.self__attn_0.source
           else:
               return attention_module.source.self__attn_0.source.self_attn_dropout_0

   model = StandardizedTransformer(
       "yujiepan/gptj-tiny-random",
       rename_config=RenameConfig(attn_prob_source=GPTJAttnProbFunction())
   )

Test the implementation:

.. code-block:: python

   with model.trace("Hello world"):
       attn_probs = model.attention_probabilities[0].save()
       # Verify shape: (batch, heads, seq_len, seq_len)
       # Verify last dimension sums to 1
       assert attn_probs.sum(dim=-1).allclose(torch.ones_like(attn_probs.sum(dim=-1)))

Attention Probabilities (Optional)
-----------------------------------

Only implement attention probabilities if you need them for your research. The process requires:

1. **Find the attention weights**: Use ``model.scan()`` to explore the forward pass
2. **Locate the hook point**: Find where attention probabilities are computed (usually after dropout)
3. **Create AttnProbFunction**: Implement the hook location
4. **Test thoroughly**: Verify shape and normalization

Key considerations:

- Use ``scan()`` first, fall back to ``trace()`` if needed
- Hook after dropout but before multiplication/masking when possible
- Avoid hooks inside conditional statements
- Test with dummy inputs to verify tensor shapes

Troubleshooting
---------------

Common issues and solutions:

**"Could not find layers module"**
   Set ``layers_name`` in ``RenameConfig``

**"Could not find ln_final module"**
   Set ``ln_final_name`` in ``RenameConfig``

**"Attention probabilities test failed"**
   Either disable attention probabilities or implement ``AttnProbFunction``

**Shape mismatches**
   Check if your model's MLP returns tuples: ``mlp_returns_tuple=True``

Testing Your Configuration
--------------------------

``nnterp`` automatically validates your configuration:

.. code-block:: python

   # This will run automatic tests
   model = StandardizedTransformer("your-model", rename_config=config)
   
   # Manual validation
   with model.trace("test"):
       # Check layer I/O shapes
       layer_out = model.layers_output[0]
       assert layer_out.shape == (batch_size, seq_len, hidden_size)
       
       # Check attention probabilities if enabled
       if model.attention_probabilities.enabled:
           attn_probs = model.attention_probabilities[0]
           assert attn_probs.shape == (batch_size, num_heads, seq_len, seq_len)

The tests verify:

- Module naming correctness
- Tensor shapes at each layer
- Attention probabilities normalization (if enabled)
- I/O compatibility with nnterp's accessors

Once your model loads successfully, all ``nnterp`` features become available with the standard interface.