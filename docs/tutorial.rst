.. _ref-tutorial:

========
Tutorial
========

Django-inviteme is a reusable app that relies on its own code and doesn't require any other extra app.


 Installation
============

Installing Django-inviteme is as simple as checking out the source and adding it to your project or ``PYTHONPATH``.

Use git, pip or easy_install to check out Django-inviteme from Github_ or get a release from PyPI_:

  1. Use **git** to clone the repository, and then install the package (read more about git_):

    * ``git clone git://github.com/danirus/django-inviteme.git`` and

    * ``python setup.py install``

  2. Or use **pip** (read more about pip_):

    * Do ``pip install django-inviteme``, or

    * Edit your project's ``requirements`` file and append either the Github_ URL or the package name ``django-inviteme``, and then do ``pip install -r requirements``.

  3. Or use **easy_install** (read more about easy_install_): 

    * Do ``easy_install django-inviteme``


.. _Github: http://github.com/danirus/django-inviteme
.. _PyPI: http://pypi.python.org/
.. _pip: http://www.pip-installer.org/
.. _easy_install: http://packages.python.org/distribute/easy_install.html
.. _git: http://git-scm.com/


Configuration
=============

1. Add ``'inviteme'`` to your ``INSTALLED_APPS`` setting.

2. Add ``url(r'^invite/', include('inviteme.urls'))`` to your ``urls.py``.

3. Create a ``inviteme`` directory in your templates directory and copy the default templates from django-inviteme into the new created directory.

4. Run ``python manage.py syncdb`` that creates the ``inviteme_contact_mail`` table.


Customization
-------------

1. Optionally you can add some settings to control Django-contactme behaviour (see :doc:`settings`), but they all have sane defaults.

2. Customize the templates (see :doc:`templates`) in your ``inviteme`` templates directory to make them fit in your design.


.. _workflow-label:

Workflow
========

Workflow described in 3 actions:

1. Get the Contact Form.

 #. Render the Contact Form page. Omit this at will by using the ``render-mail-form`` templatetag (see :doc:`templatetags`) in your own templates.

2. Post the Contact Form.

 #. Check if there are *form security errors*. Django_ContactMe forms are protected with ``timestamp``, ``security_hash`` and ``honeypot`` field, following the same approach as the built-in `Django Comments Framework <https://docs.djangoproject.com/en/1.3/ref/contrib/comments/>`_. In case of *form security errors* send a 400 code response and stop.

 #. Check whether there are other *form errors* (basically check if the ``email`` field is empty). In such a case render the *Mail Form* again, with the *form errors* and stop.

 #. Send signal ``inviteme.signals.confirmation_will_be_requested``. If any receiver returns ``False``, send a discarded response to the user and stop.

 #. Send a confirmation email to the user with a confirmation URL.

 #. Send signal ``inviteme.signals.confirmation_requested``.

 #. Render a *"confirmation has been sent to you by email"* template.

3. Visit the Confirmation URL.

 #. Check whether the token in the confirmation URL is correct. If it isn't raise a 404 code response and stop.

 #. Create a ``ContactMail`` model instance with the email address secured in the URL.

 #. Send signal ``confirmation_received``. If any receiver return False, send a discarded response to the user and stop.

 #. Send an email to ``settings.INVITEME_NOTIFY_TO`` addresses indicating that a new invitation request has been received.

 #. Render a *"your invitation request has been received, thank you"* template.


Creating the secure token for the confirmation URL
--------------------------------------------------

The Confirmation URL sent by email to the user has a secured token with the contact form data. To create the token Django-ContactMe uses the module ``signed.py`` authored by Simon Willison and provided in `Django-OpenID <http://github.com/simonw/django-openid>`_. 

``django_openid.signed`` offers two high level functions:

* **dumps**: Returns URL-safe, sha1 signed base64 compressed pickle of a given object.

* **loads**: Reverse of dumps(), raises ValueError if signature fails.

A brief example::

    >>> signed.dumps("hello")
    'UydoZWxsbycKcDAKLg.QLtjWHYe7udYuZeQyLlafPqAx1E'

    >>> signed.loads('UydoZWxsbycKcDAKLg.QLtjWHYe7udYuZeQyLlafPqAx1E')
    'hello'

    >>> signed.loads('UydoZWxsbycKcDAKLg.QLtjWHYe7udYuZeQyLlafPqAx1E-modified')
    BadSignature: Signature failed: QLtjWHYe7udYuZeQyLlafPqAx1E-modified


There are two components in dump's output ``UydoZWxsbycKcDAKLg.QLtjWHYe7udYuZeQyLlafPqAx1E``, separatad by a '.'. The first component is a URLsafe base64 encoded pickle of the object passed to dumps(). The second component is a base64 encoded hmac/SHA1 hash of "$first_component.$secret".

Calling signed.loads(s) checks the signature BEFORE unpickling the object -this protects against malformed pickle attacks. If the signature fails, a ValueError subclass is raised (actually a BadSignature).


.. _signals-and-receivers-label:

Signals and receivers
=====================

The workflow mentions that django-inviteme sends 3 signals:

#. **confirmation_will_be_requested**: Sent just before a confirmation message is requested.

#. **confirmation_requested**: Sent just after a confirmation message is requested.

#. **confirmation_received**: Sent just after a confirmation has been received.

See :doc:`signals` to know more.

You may want to extend django-inviteme by registering a receiver for any of this signals. 

An example function receiver might check the datetime a user submitted a contact message and the datetime the confirmation URL has been clicked. If the difference between them is over 7 days the message could be discarded with a graceful `"sorry, too old message"` template.

Extending the demo site with the following code would do the job::

    #----------------------------------------
    # append the code below to demo/views.py:

    from datetime import datetime, timedelta
    from inviteme import signals

    def check_submit_date_is_within_last_7days(sender, data, request, **kwargs):
	plus7days = timedelta(days=7)
	if data["submit_date"] + plus7days < datetime.now():
	    return False
    signals.confirmation_received.connect(check_submit_date_is_within_last_7days)
    
    
    #-----------------------------------------------------
    # change get_instance_data in inviteme/forms.py to cheat a bit and 
    # make Django believe that the contact form was submitted 7 days ago:

    def get_instance_data(self):
        """
        Returns the dict of data to be used to create a contact message. 
        """
	from datetime import timedelta                                 # ADD THIS

        return dict(
            email       = self.cleaned_data["email"],
    #        submit_date = datetime.datetime.now(),                    # COMMENT THIS
            submit_date = datetime.datetime.now() - timedelta(days=8), # ADD THIS
        )

Try the demo site again and see that the `inviteme/discarded.html` template is rendered after clicking on the confirmation URL.
