Customize a New One-shot Trainer
================================

One-shot trainers should inherit ``nni.retiarii.oneshot.BaseOneShotTrainer``, and need to implement ``fit()`` (used to conduct the fitting and searching process) and ``export()`` method (used to return the searched best architecture).

Writing a one-shot trainer is very different to single-arch evaluator. First of all, there are no more restrictions on init method arguments, any Python arguments are acceptable. Secondly, the model fed into one-shot trainers might be a model with Retiarii-specific modules, such as LayerChoice and InputChoice. Such model cannot directly forward-propagate and trainers need to decide how to handle those modules.

A typical example is DartsTrainer, where learnable-parameters are used to combine multiple choices in LayerChoice. Retiarii provides ease-to-use utility functions for module-replace purposes, namely ``replace_layer_choice``, ``replace_input_choice``. A simplified example is as follows: 

.. code-block:: python

    from nni.retiarii.oneshot import BaseOneShotTrainer
    from nni.retiarii.oneshot.pytorch import replace_layer_choice, replace_input_choice


    class DartsLayerChoice(nn.Module):
        def __init__(self, layer_choice):
            super(DartsLayerChoice, self).__init__()
            self.name = layer_choice.label
            self.op_choices = nn.ModuleDict(layer_choice.named_children())
            self.alpha = nn.Parameter(torch.randn(len(self.op_choices)) * 1e-3)

        def forward(self, *args, **kwargs):
            op_results = torch.stack([op(*args, **kwargs) for op in self.op_choices.values()])
            alpha_shape = [-1] + [1] * (len(op_results.size()) - 1)
            return torch.sum(op_results * F.softmax(self.alpha, -1).view(*alpha_shape), 0)


    class DartsTrainer(BaseOneShotTrainer):

        def __init__(self, model, loss, metrics, optimizer):
            self.model = model
            self.loss = loss
            self.metrics = metrics
            self.num_epochs = 10

            self.nas_modules = []
            replace_layer_choice(self.model, DartsLayerChoice, self.nas_modules)

            ... # init dataloaders and optimizers

        def fit(self):
            for i in range(self.num_epochs):
                for (trn_X, trn_y), (val_X, val_y) in zip(self.train_loader, self.valid_loader):
                    self.train_architecture(val_X, val_y)
                    self.train_model_weight(trn_X, trn_y)

        @torch.no_grad()
        def export(self):
            result = dict()
            for name, module in self.nas_modules:
                if name not in result:
                    result[name] = select_best_of_module(module)
            return result

The full code of DartsTrainer is available to Retiarii source code. Please have a check at :githublink:`DartsTrainer <nni/retiarii/oneshot/pytorch/darts.py>`.
