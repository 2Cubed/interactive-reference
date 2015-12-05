The Analyst's Field Guide
*************************

Intro
=====

Multiple types of analyses can be performed on Tetris controls. They can be requested in the controls object by putting their corresponding ID (see below) in the control's associated "analysis" array. See :doc:`/games` for more information about the controls structure.

==== =============
 ID   Analysis
==== =============
0    Frequency
1    Mean
2    Standard Deviation
3    Quartiles
==== =============

A Note on Influence
-------------------

Elsewhere in this documentation and on the Beam website, you may have seen reference to users' "influence". Essentially what this is is a multiplier on the weight of that user's input in the analysis. Each user starts with a base influence of 1, and increase in influence linearly increases their input's weight. For example, a user with an influence of ``1.1`` would be weighted 10% higher than the average user when calculating the mean than a user with ``1.0`` influence.

Notation
--------

Mathematical representations are provided below. Some symbols you will see are:

- :math:`n` - the number of users
- :math:`\alpha_i` - the scalar input provided by the ith user.
- :math:`\beta_i` - the influence of the ith user.

Frequency
=========

Frequency is the most fundamental sort of analysis we provide. It's simply a sum of all occurrences of an action, such as a key press.

.. math::

    f = \sum_{i = 0}^n \alpha_i \beta_i


Recipe: Let users vote on an action to take
-------------------------------------------

One of the classical interactions with livestreamed gaming has been to effect the most popular action. This can be easily done using frequency analysis alone. An algorithm to do so might look like the following:

 1. Show a message on the screen telling the users to vote for something. Set a timer.
 2. Initialize an empty hashmap of uints to uints.
 3. Wait for either report, or the countdown to expire:

    - case report:

        a. Pop the first tactile input off the report.
        b. If the tactile's key isn't in the hashmap, initialize its value to zero.
        c. Add the tactile's reported frequency to its associated hashmap value.
        d. If there's still more inputs to process, goto a.

    - case timer:

        a. Find the item in the hashmap with the greatest value.
        b. That's what the users voted to do. Do it!

Mean
====

Mean is the average of inputs given by users, weighted by their influence. The calculation can be represented with the following equation:

.. math::

    \mu = \frac{f}{\sum_{i = 0}^n \beta_i}

Recipe: Only attack a monster when enough users vote on the action
------------------------------------------------------------------

The mean provides an easy way to activate a button press after a threshold is reached. An algorithm might look like so:

 1. Define a threshold value of 0.5.
 2. Wait for a report to come in.
 3. Divide the mean of each key by the report's quorum.
 4. If any calculated values are over 0.5, fire a key press.
 5. Goto 2.

Standard Deviation
==================

The standard deviation allows you to determine how "focused" users are on a particular action. It's calculated using the following rather daunting equation: (`source <http://www.itl.nist.gov/div898/software/dataplot/refman2/ch2/weightsd.pdf>`_)

.. math::

    \sigma = \sqrt{\frac{\sum_{i = 0}^n \beta_i (\alpha_i - \mu)^2}{\frac{n - 1}{n} \sum_{i = 0}^n \beta_i}}


Recipe: Doing some Minecraft parkour
------------------------------------

Making use of the standard deviation can allow a more rapid decision-making process than a simple interval-based voting mechanism, and additionally it allows people to vote on scalar actions using the joystick. In Minecraft parkour, we want to be really sure of an action before taking it, so we can use standard deviation on joysticks to only move when we're sure of the direction to go:

 1. Wait for a report to come in.
 2. If the standard deviation of the joysticks is not sufficiently small, Goto 1.
 3. Otherwise, move a block in the chosen direction!

