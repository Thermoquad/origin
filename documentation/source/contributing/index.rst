Contributing
############

.. note::
   This documentation is under construction.

How to Contribute
*****************

Thermoquad welcomes contributions! Here's how to get started.

Code Style
**********

* Use ``snake_case`` for functions and variables
* Use ``UPPER_CASE`` for constants and macros
* Run ``task format`` before committing

Git Workflow
************

1. Create a feature branch
2. Make your changes
3. Run tests with ``task test``
4. Submit a pull request

Commit Messages
***************

Use conventional commit format:

.. code-block:: text

   type(scope): subject

   body

Types: feat, fix, docs, style, refactor, test, chore

Documentation
*************

Documentation is built with Sphinx. To build locally:

.. code-block:: bash

   cd origin/documentation
   task html
   task serve
