---
title: LIT - Setup Guide
layout: layouts/sub.liquid

hero-height: 245
hero-image: /assets/images/LIT_FAQs_Banner.png
hero-title: "LIT is simple to use"
hero-copy: "Get up and running quickly, and run the demos or run LIT on your own models and data"

sub-nav: '<a href="#install">Installation</a><a href="#demos">Run demos</a><a href="#custom">LIT with your own models and data</a>'
color: "#fef0f7"
---

<div class="mdl-cell--8-col mdl-cell--8-col-tablet mdl-cell--4-col-phone">

<a name="install"></a>

# Install LIT

LIT can be installed via pip, or can be built from source.

## Install via pip

```bash
pip install lit-nlp
```

The pip installation will install all necessary prerequisite packages for use of the core LIT package. It also installs the code to run our demo examples. It does not install the prerequisites for those demos, so you need to install those yourself if you wish to run the demos. To install those, we recommend using conda with our included [environment.yml](https://github.com/PAIR-code/lit/blob/main/environment.yml).

```bash
# Set up Python environment
cd ~/lit
conda env create -f environment.yml
conda activate lit-nlp
conda install cudnn cupti  # optional, for GPU support
conda install -c pytorch pytorch  # optional, for PyTorch
```

If you want to update any of the frontend or core code, you can install a local copy from source:

## Install from source

Download the code from our [GitHub repo](https://github.com/PAIR-code/lit/) and set up a Python environment:

```bash
git clone https://github.com/PAIR-code/lit.git ~/lit

# Set up Python environment
cd ~/lit
conda env create -f environment.yml
conda activate lit-nlp
conda install cudnn cupti  # optional, for GPU support
conda install -c pytorch pytorch  # optional, for PyTorch

# Build the frontend
pushd lit_nlp; yarn && yarn build; popd
```

{% include partials/spacer height:50 %}

<a name="demos"></a>

# Run the included demos

LIT ships with a number of demos that can easily be run after installation.

## Sentiment classifier

```bash
python -m lit_nlp.examples.quickstart_sst_demo --port=5432
```

This will fine-tune a [BERT-tiny](https://arxiv.org/abs/1908.08962) model on the [Stanford Sentiment Treebank](https://nlp.stanford.edu/sentiment/treebank.html), which should take less than 5 minutes on a GPU.
After training completes, it'll start a LIT server on the development set.
Navigate to http://localhost:5432 for the UI.

## Language modeling

```bash
python -m lit_nlp.examples.pretrained_lm_demo --models=bert-base-uncased --port=5432
```

In this demo, you can explore predictions from a pretrained language model (i.e. fill in the blanks).
Navigate to http://localhost:5432 for the UI.

## More examples
The [examples](https://github.com/PAIR-code/lit/tree/main/lit_nlp/examples) directory contains additional examples to explore, all of which can be run similarly to those above.

{% include partials/spacer height:50 %}

<a name="custom"></a>

# Use LIT on your own models and data

This is a brief overview of how to run LIT with your own models and datasets.
For more details, see the documentation in the [code repository](https://github.com/pair-code/lit).

To run LIT with your own models and data, you can create a custom `demo.py`
script that passes these to the LIT server. For example:

```py
def main(_):
  # MulitiNLIData implements the Dataset API
  datasets = {
      'mnli_matched': MultiNLIData('/path/to/dev_matched.tsv'),
      'mnli_mismatched': MultiNLIData('/path/to/dev_mismatched.tsv'),
  }

  # NLIModel implements the Model API
  models = {
      'model_foo': NLIModel('/path/to/model/foo/files'),
      'model_bar': NLIModel('/path/to/model/bar/files'),
  }

  lit_demo = lit_nlp.dev_server.Server(models, datasets, port=4321)
  return lit_demo.serve()

if __name__ == '__main__':
  main()
```

Conceptually, a dataset is just a list of examples and a model is just a
function that takes examples and returns predictions. The [`Dataset`](#datasets)
and [`Model`](#models) classes implement this, and provide metadata to describe themselves to other
components.

For full examples, see
[examples](https://github.com/PAIR-code/lit/tree/main/lit_nlp/examples). In particular:

*   [`simple_tf2_demo.py`](https://github.com/PAIR-code/lit/tree/main/lit_nlp/examples/simple_tf2_demo.py)
    for a self-contained Keras/TF2 model for sentiment analysis.
*   [`simple_pytorch_demo.py`](https://github.com/PAIR-code/lit/tree/main/lit_nlp/examples/simple_pytorch_demo.py)
    for a self-contained PyTorch model for sentiment analysis.

<a name="datasets"></a>
## Datasets

Datasets ([`Dataset`](https://github.com/PAIR-code/lit/tree/main/lit_nlp/api/dataset.py)) are
just a list of examples, with associated type information following LIT's type system.

*   `spec()` should return a flat dict that describes the fields in each example
*   `self._examples` should be a list of flat dicts

Implementations should subclass
[`Dataset`](https://github.com/PAIR-code/lit/tree/main/lit_nlp/api/dataset.py). Usually this
is just a few lines of code - for example, the following is a complete dataset
loader for [MultiNLI](https://cims.nyu.edu/~sbowman/multinli/):

```py
class MultiNLIData(Dataset):
  """Loader for MultiNLI development set."""

  NLI_LABELS = ['entailment', 'neutral', 'contradiction']

  def __init__(self, path):
    # Read the eval set from a .tsv file as distributed with the GLUE benchmark.
    df = pandas.read_csv(path, sep='\t')
    # Store as a list of dicts, conforming to self.spec()
    self._examples = [{
      'premise': row['sentence1'],
      'hypothesis': row['sentence2'],
      'label': row['gold_label'],
      'genre': row['genre'],
    } for _, row in df.iterrows()]

  def spec(self):
    return {
      'premise': lit_types.TextSegment(),
      'hypothesis': lit_types.TextSegment(),
      'label': lit_types.Label(vocab=self.NLI_LABELS),
      # We can include additional fields, which don't have to be used by the model.
      'genre': lit_types.Label(),
    }
```

This implementation uses Pandas to read a TSV file, but you can also use
services like [TensorFlow Datasets](https://www.tensorflow.org/datasets) -
simply wrap them in your `__init__()` function.

Note that you can freely add additional features - such as `genre` in the
example above - which the model may not be aware of. The LIT UI will recognize
these features for slicing, binning, etc., and they will also be available to
interpretation components such as custom metrics.

<a name="models"></a>
## Models

Models ([`Model`](https://github.com/PAIR-code/lit/tree/main/lit_nlp/api/model.py)) are
functions which take inputs and produce outputs, with associated type
information following LIT's type system. The core
API consists of three methods:

*   `input_spec()` should return a flat dict that describes necessary input
    fields
*   `output_spec()` should return a flat dict that describes the model's
    predictions and any additional outputs
*   `predict_minibatch()` and/or `predict()` should take a sequence of inputs
    (satisfying `input_spec()`) and yields a parallel sequence of outputs
    matching `output_spec()`.

Implementations should subclass
[`Model`](https://github.com/PAIR-code/lit/tree/main/lit_nlp/api/model.py). An example for
[MultiNLI](https://cims.nyu.edu/~sbowman/multinli/) might look something like:

```py
class NLIModel(Model):
  """Wrapper for a Natural Language Inference model."""

  NLI_LABELS = ['entailment', 'neutral', 'contradiction']

  def __init__(self, model_path, **kw):
    # Load the model into memory so we're ready for interactive use.
    self._model = _load_my_model(model_path, **kw)

  ##
  # LIT API implementations
  def predict(self, inputs: List[Input]) -> Iterable[Preds]:
    """Predict on a single minibatch of examples."""
    examples = [self._model.convert_dict_input(d) for d in inputs]  # any custom preprocessing
    return self._model.predict_examples(examples)  # returns a dict for each input

  def input_spec(self):
    """Describe the inputs to the model."""
    return {
        'premise': lit_types.TextSegment(),
        'hypothesis': lit_types.TextSegment(),
    }

  def output_spec(self):
    """Describe the model outputs."""
    return {
      # The 'parent' keyword tells LIT where to look for gold labels when computing metrics.
      'probas': lit_types.MulticlassPreds(vocab=NLI_LABELS, parent='label'),
    }
```

Unlike the dataset example, this model implementation is incomplete - you'll
need to customize `predict()` (or `predict_minibatch()`) accordingly with any
pre- or post-processing needed, such as tokenization.

Note: The `Model` base class implements simple batching, aided by the
`max_minibatch_size()` function. This is purely for convenience, since most deep
learning models will want this behavior. But if you don't need it, you can
simply override the `predict()` function directly and handle large inputs
accordingly.

Note: there are a few additional methods in the model API - see
[`Model`](https://github.com/PAIR-code/lit/tree/main/lit_nlp/api/model.py) for details.

</div>