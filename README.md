# CleverDict

![simplicity](https://image.slidesharecdn.com/iotshifts20150911-151021225240-lva1-app6891/95/smart-citizens-populating-smart-cities-iotshifts-19-638.jpg?cb=1506979421)

## OVERVIEW

```CleverDict``` is a hybrid Python data class which allows both ```object.attribute``` and ```dictionary['key']``` notation to be used simultaneously and interchangeably.  It's particularly handy when your code is mainly object-orientated but you want a 'DRY' and extensible way to import data in json/dictionary format into your objects... or vice versa... without having to write extra code just to handle the translation.

The class also optionally triggers a ```._save()``` method (which you can adapt or overwrite) which it calls whenever an attribute or dictionary value is created or changed.  This is especially useful if you want your object's values to be automatically pickled, encoded, saved to a file or database, uploaded to the cloud etc. without having to slavishly call your update function after every single operation where attributes (might) change.


## INSTALLATION
No dependencies.  Very lightweight:

    pip install cleverdict

or to cover all bases...

    python -m pip install cleverdict --upgrade --user

## QUICKSTART

```CleverDict``` objects behave like normal Python dictionaries, but with the convenience of immediately offering read and write access to their data (keys and values) using the ```object.attribute``` syntax, which many people find easier to type and more intuitive to read and understand.

### 1. BASIC USE

    >>> from cleverdict import CleverDict
    >>> x = CleverDict({'total':6, 'usergroup': "Knights of Ni"})

    >>> x.total
    6
    >>> x['total']
    6
    >>> x.usergroup
    'Knights of Ni'
    >>> x['usergroup']
    'Knights of Ni'

### 2. DATA FROM OTHER OBJECTS
You can import an existing object's data (but not its methods) directly using ```vars()```:

    x = CleverDict(vars(my_existing_object))

    >>> list(x.items())
    [('total', 6), ('usergroup', 'Knights of Ni'), ('life', 42)]


### 3. KEYWORD ARGUMENTS
You can also supply keyword arguments like this:

    >>> x = CleverDict(created = "today", review = "tomorrow")

    >>> x.created
    'today'
    >>> x['review']
    'tomorrow'

Or using ```.fromkeys()``` like this:

    >>> x = CleverDict().fromkeys(["Abigail", "Tino", "Isaac"], "Year 9")

    >>> x
    CleverDict([('Abigail', 'Year 9'), ('Tino', 'Year 9'), ('Isaac', 'Year 9')])

Regardless of which syntax you use, new values are immediately available via both methods:

    >>> x['life'] = 42
    >>> x.life += 1
    >>> x['life']
    43

    >>> del x['life']
    >>> x.life
    # KeyError: 'life'


### 4. ATTRIBUTE NAMES AND ALIASES

By default ```CleverDict``` tries to find valid attribute names for dictionary keys which would otherwise fail.  This includes keywords, null strings, most punctuation marks, and keys starting with a numeral.  So for example ```7``` (integer) becomes ```"_7"``` (string):

    >>> x = CleverDict({7: "Seven"})

    >>> x._7
    'Seven'
    >>> x
    CleverDict({'_7':'Seven'})

```CleverDict``` keeps the original dictionary keys and values unchanged in ```.data``` (a feature it inherits from ```collections.UserDict```), and remembers any normalised attribute names as aliases in ```._alias```.  You can view a nice summary of all attributes and alias just by using ```print()``` or by accessing an object's ```.__str__``` method:

    >>> x=CleverDict({True: "the truth"})
    >>> x.data
    {True: 'the truth'}
    >>> print(x)
    CleverDict
        x[7] == x['_7'] == x._7 == 'Seven'

You can toggle normalisation Off and On (for all future operations on all current and future objects of the class) as follows:

    >>> CleverDict.normalise = False  # or: True
    >>> x = CleverDict({1: "One"})

    >>> hasattr(x, "_1")
    False
    >>> x
    CleverDict({1:'One'})
    >>> x[1]
    'One'

**NB** Most unicode letters are valid in attribute names since [PEP3131](https://www.python.org/dev/peps/pep-3131/), but ```CleverDict``` replaces other invalid characters such as punctuation marks with "```_```" on a first come, first served basis.  This can result in an ```AttributeError``` (which you can get round by renaming the offending dictionary keys).  For example:

    >>> x = CleverDict({"one-two": "hypen", "one/two": "forward slash"})
    AttributeError: duplicate alias already exists for 'one-two'

Normalisation can get even more confusing in certain edge cases... For example not everyone knows that when creating a dictionary, the keys ```0```, ```0.0```, and ```False``` are considered the same in Python.  Likewise ```1```, ```1.0```, and ```True```, and ```1234``` and ```1234.0```.  Also if you create a regular dictionary using more than one of these different identities, they will appear to 'overwrite' each other keeping the first Key specified but the last Value specified:


    x = {1: "one", True: "the truth"}
    >>> x
    {1: 'the truth'}

You'll be relieved to know ```CleverDict``` handles these cases but we thought it was worth mentioning in case you came across them first and wondered what the heck was going on (as I did initially)!  You can always work out what's going on 'under the hood' by looking at the aliases with ```print()```.


### 5. ENABLING THE AUTO-SAVE FUNCTION
You can set pretty much any function to run automatically whenever a ```CleverDict``` value is created or changed.  There's an example function in ```cleverict.test_cleverdict``` which demonstrates this:

    >>> from cleverdict.test_cleverdict import my_example_save_function
    >>> CleverDict._save = my_example_save_function

    >>> x = CleverDict({'total':6, 'usergroup': "Knights of Ni"})
    Notional save to database: .total = 6 <class 'int'>
    Notional save to database: .usergroup = Knights of Ni <class 'str'>

    >>> x.life = 42
    Notional save to database: .life = 42 <class 'int'>


The example function above also appends output to a file, which you might want for debugging, auditing,  further analysis etc.:

    >>> with open("example.log","r") as file:
    ...     log = file.read()

    >>> log.splitlines()
    ["Notional save to database: .age = 10 <class 'int'>",
    "Notional save to database: .total = 6 <class 'int'>",
    "Notional save to database: .usergroup = Knights of Ni <class 'str'>"]

**NB**: The ```._save()``` method is a *class* method, so changing ```CleverDict._save``` will apply the new ```._save()``` method to all previously created ```CleverDict``` objects as well as future ones.


### 6. CREATING YOUR OWN AUTO-SAVE FUNCTION
When writing your own ```._save()``` function, you'll need to specify three arguments as follows:


    >>> def your_function(self, name: str = "", value: any = ""):
    ...     print("Ni!")


* **self**: because we're dealing with objects and classes...
* **name**: a valid Python ```.attribute``` name preferably, otherwise you'll only be able to access it using ```dictionary['key']``` notation later on.
* **value**: anything

### 7. SETTING DIFFERENT AUTO-SAVE FUNCTIONS FOR DIFFERENT OBJECTS
If you want to specify different ```._save()``` behaviours for different objects, consider creating sublasses that inherit from ```CleverDict``` and set a different
```._save()``` function for each subclass e.g.:

    >>> class Type1(CleverDict): pass
    >>> Type1._save = my_save_function1

    >>> class Type2(CleverDict): pass
    >>> Type2._save = my_save_function2


### 8. SETTING AN ATTRIBUTE WITHOUT CREATING A DICTIONARY ITEM
We've included the ```.setattr_direct()``` method in case you want to set an object attribute *without* creating the corresponding dictionary key/value.  This could be useful for storing save data for example, and is used internally to store aliases in ```.alias```.

Here's one way you could create a .store attribute and customise the auto-save behaviour:

    class SaveDict(CleverDict):
        def __init__(self, *args, **kwargs):
            self.setattr_direct('store', [])
            super().__init__(*args, **kwargs)

        def save(self, name, value):
            self.store.append((name, value))

## CONTRIBUTING

We'd love to see Pull Requests (and relevant tests) from other contributors, particularly if you can help with the following:

1. It would be great if ```CleverDict``` behaviour could be easily 'grafted on' to existing classes using inheritance, without causing recursion or requiring a rewrite/overwrite of the original class.

    For example if it were as easy as:

    ```
    >>> class MyDatetime(datetime.datetime, CleverDict):
    ...     pass

    >>> mdt = MyDatetime.now()
    >>> mdt.hour
    4
    >>> mdt['hour']
    4
    ```

2. Complete set of aliases for True/False in ```__str__```.  Given that ```1```, ```1.0```, and ```True``` are considered equivalent as dictionary keys, it would be helpfully explicit to create all possibilities as aliases regardless of which Key is given initially.  Likewise ```False```.
    ```
    >>> x = CleverDict({True: "Is this?"})
    >>> print(x)
    CleverDict
        x[True] == x['_True'] == x._True == x[1] == x[1.0] == 'Is this?'
    ```

3. The ```.data``` attribute is reserved by ```UserDict``` and it shouldn't be possible to overwrite/break it, but it is currently, but ```test_data_attribute()``` fails consistently.  Also since ```data``` is a relatively common Key/Attribute name it's likely to feature in people's existing dictionaries or objects.  I don't know how, but wonder if we can change the (inherited) behaviour of ```UserDict``` such that it accesses ```_data``` rather than ```data```, which would also be consistent with our use of ```_alias```.  I tried various ways to change this in both in ```normalise()``` and in the main class, but got into a recursion tangle!

4.  A nice feature enhancement to ```CleverDict``` would be to offer a method for setting aliases directly; once the ```_alias``` dictionary is updated (I think) the existing functionality would automatically update all aliases whenver values change, which would be fantastic.


## CREDITS
```CleverDict``` was developed jointly by Peter Fison, Ruud van der Ham, Loic Domaigne, and Rik Huygen who met on the friendly and excellent Pythonista Cafe forum (www.pythonistacafe.com).  Peter got the ball rolling after noticing a super-convenient, but not fully-fledged feature in Pandas that allows you to (mostly) use ```object.attribute``` syntax or ```dictionary['key']``` syntax interchangeably. Ruud, Loic and Rik then started swapping ideas for a hybrid  dictionary/data class based on ```UserDict``` and the magic of ```__getattr__``` and ```__setattr__```, and ```CleverDict``` was born*.

>(\*) ```CleverDict``` was originally called ```attr_dict``` but several confusing flavours of this and ```AttrDict``` exist on PyPi and Github already.  Hopefully the new name raises a wry smile as well as being more memorable...
