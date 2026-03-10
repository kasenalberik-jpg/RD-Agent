=============================
Strategy notes (research)
=============================

This page captures how RD-Agent uses the word *strategy* in code, focusing on the
core evolutionary loop (``EvolvingStrategy`` / ``RAGStrategy``) and the main
CoSTEER implementation.


Core abstractions
-----------------

The main abstractions live in ``rdagent/core/evolving_framework.py``:

* ``EvolvableSubjects``: the object being evolved (deep-clonable, evaluable).
* ``EvoStep``: a single iteration record containing:

  * ``evolvable_subjects`` (the current solution state)
  * optional ``queried_knowledge`` (from RAG)
  * optional ``feedback`` (from evaluation)

* ``EvolvingStrategy``: defines *how* an ``EvolvableSubjects`` instance is
  modified over time via ``evolve_iter(...)``. The generator form allows an
  evolving process to yield partial implementations (that can be evaluated
  mid-way).

* ``RAGStrategy``: defines retrieval and knowledge base maintenance. It is
  responsible for querying knowledge given the current solution and trace, and
  (optionally) generating + persisting new knowledge.


Runtime control-flow (RAGEvoAgent)
----------------------------------

The coordination logic is in ``rdagent/core/evolving_agent.py`` (``RAGEvoAgent``):

1. (Optional) query knowledge via ``rag.query(evo, evolving_trace)``.
2. evolve via ``evolving_strategy.evolve_iter(...)``.
3. evaluate each yielded partial solution via ``RAGEvaluator.evaluate_iter(...)``.
4. package the end-of-loop result as an ``EvoStep`` and append it to
   ``evolving_trace``.
5. (Optional) let ``RAGStrategy`` self-generate and dump knowledge base updates.

Key design point: both evolving and evaluation assume *in-place modification* of
the evolving object.


CoSTEER: MultiProcessEvolvingStrategy
------------------------------------

The CoSTEER multi-task implementation is in
``rdagent/components/coder/CoSTEER/evolving_strategy.py``.

``MultiProcessEvolvingStrategy`` is an ``EvolvingStrategy`` specialized for an
``EvolvingItem`` that contains multiple sub-tasks. Its high-level behavior:

* Uses ``CoSTEERQueriedKnowledge`` to:

  * short-circuit tasks already marked as successful (reuse implementations)
  * avoid tasks marked as failed (and optionally skip tasks in *improve_mode*)

* Implements scheduled tasks in parallel using ``multiprocessing_wrapper``.
* Applies returned modifications back onto per-task workspaces via
  ``assign_code_list_to_evo(...)``.

Concrete subclasses (examples):

* ``FactorMultiProcessEvolvingStrategy``
  (``rdagent/components/coder/factor_coder/evolving_strategy.py``)
* ``ModelMultiProcessEvolvingStrategy``
  (``rdagent/components/coder/model_coder/evolving_strategy.py``)

These specialize ``implement_one_task(...)`` and how generated code is injected
into the appropriate workspace.


Other strategy-style components
-------------------------------

Not all "strategy" classes participate in the evolutionary loop. For example,
data-science proposal generation contains a separate strategy interface:

* ``DiversityContextStrategy``
  (``rdagent/scenarios/data_science/proposal/exp_gen/diversity_strategy.py``)

It controls *when* to inject cross-trace diversity context during experiment
generation (e.g., only at root, until SOTA is achieved, always).


Extension checklist (adding a new evolving strategy)
---------------------------------------------------

1. Define/choose an ``EvolvableSubjects`` implementation for your scenario.
2. Implement ``EvolvingStrategy.evolve_iter(...)``:

   * accept ``queried_knowledge`` and ``evolving_trace`` (if used)
   * yield after each meaningful partial implementation

3. Provide a matching evaluator:

   * simple: implement ``Evaluator.evaluate``
   * iterative: implement ``RAGEvaluator.evaluate_iter`` if you want mid-step
     feedback

4. If you want retrieval/self-learning:

   * implement a ``RAGStrategy`` and wire it into ``RAGEvoAgent``.


Notes / sharp edges (observed in current code)
----------------------------------------------

These are not necessarily bugs, but are useful when extending/debugging:

* Avoid mutable defaults in new strategy code. (Some existing implementations
  use ``evolving_trace=[]`` in signatures; prefer ``None`` + initialization.)
* Keep ``implement_one_task`` outputs consistent within a strategy family.
  (Some current coder strategies return ``str`` in one case and ``dict`` in
  another, requiring adapter logic in ``assign_code_list_to_evo``.)
* ``EvolvingStrategy`` is an ABC requiring ``evolve_iter``. Classes that only
  define an ``evolve`` method are not concrete strategies unless they also
  implement ``evolve_iter``.
