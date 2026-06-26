.. _pyaedt-grpc:

Running over PyAEDT (gRPC) instead of COM
=========================================

.. contents:: On this page
   :local:
   :depth: 2

pyEPR drives Ansys HFSS through a hand-rolled **COM** layer (:mod:`pyEPR.ansys`),
which is Windows-only and tied to a private scripting interface.  You can instead
have pyEPR's existing :class:`~pyEPR.DistributedAnalysis` connect through
**PyAEDT** (``ansys-aedt-core``) — Ansys's official, maintained Python API —
**entirely over gRPC, with no COM** — by passing ``use_pyaedt=True`` to
:class:`~pyEPR.ProjectInfo`.

This is additive and surgical.  When ``use_pyaedt=False`` (the default) the COM
path is byte-for-byte unchanged.  The physics is identical either way: results
match the COM path digit-for-digit (:math:`p_{mj} = 0.9755` on the demo
transmon).


Why a gRPC connection
---------------------

* **No COM / pywin32.**  The transport is gRPC, the default for AEDT since
  2022 R2, instead of COM (.NET, Windows-only).
* **Attaches to the right session.**  PyAEDT attaches to the AEDT instance that
  owns the project, avoiding the stale-session and *project-locked* errors
  common with COM's Running-Object-Table lookup.
* **Maintenance moves to Ansys.**  ``ansys-aedt-core`` is officially versioned
  and supported, rather than a private COM interface pyEPR must track by hand
  across AEDT releases.


Installation
------------

PyAEDT is an optional extra and is imported lazily — only when
``use_pyaedt=True`` — so ``import pyEPR`` never requires it:

.. code-block:: bash

   pip install "pyEPR-quantum[pyaedt]"


Usage
-----

Pass ``use_pyaedt=True`` (and, optionally, the AEDT version and session options)
to :class:`~pyEPR.ProjectInfo`.  Everything else — junctions, modes, the
analysis call — is exactly the standard pyEPR workflow.

.. code-block:: python

   import pyEPR as epr

   pinfo = epr.ProjectInfo(
       project_path=r"C:\HFSS_Projects",
       project_name="EPR_Demo_Project",
       design_name="EPR_Sample_Demo",
       setup_name="EPR_Scan",
       do_connect=False,
       use_pyaedt=True,            # <-- connect over PyAEDT / gRPC instead of COM
       aedt_version="2026.1",
   )
   pinfo.junctions["j1"] = {"Lj_variable": "Lj_Transmon", "line": "Junction_line"}

   pinfo.connect()                 # opens/attaches over gRPC

   eprd = epr.DistributedAnalysis(pinfo)
   eprd.do_EPR_analysis()          # pyEPR's normal pipeline, now over gRPC

   epra = epr.QuantumAnalysis(eprd.data_filename)
   epra.analyze_all_variations()


How it works
------------

PyAEDT's gRPC scripting handles mirror the COM scripting API, so the existing
:class:`~pyEPR.ansys.HfssDesktop` / :class:`~pyEPR.ansys.HfssProject` /
:class:`~pyEPR.ansys.HfssDesign` wrappers are reused unchanged — seeded with
``hfss.odesktop`` rather than a COM ``Dispatch`` object.  The session is flagged
``_is_grpc`` (propagated to the design), and the only behavioural change is the
field-calculator read-back:

.. list-table::
   :header-rows: 1
   :widths: 30 35 35

   * -
     - COM (default)
     - PyAEDT (``use_pyaedt=True``)
   * - connection
     - ``win32com.Dispatch``
     - ``ansys.aedt.core.Hfss`` → ``hfss.odesktop``
   * - ``CalcObject.evaluate`` read-back
     - ``ClcEval`` + ``GetTopEntryValue``
     - ``CalculatorWrite`` to a ``.fld`` file

The field-calculator token stack (``EnterQty`` / ``ClcMaterial`` / ``EnterVol``
/ ``EnterLine`` / ``Integrate``) and the eigenmode normalization
(``Solutions.EditSources``) are identical and run fine over gRPC.  The stateful
``ClcEval`` / ``GetTopEntryValue`` round-trip is the one call that does not
survive gRPC, so on a gRPC session ``evaluate`` writes the result to a file and
reads it back instead.


See also
--------

* :ref:`Tutorial 7 <tutorials>` — a full worked example.
* :mod:`pyEPR.ansys` — the COM backend, unchanged.
