Py-Qt-MVC
========

Script for auto-generating Model-View-Controller application template files for PyQt or PySide.

Manually creating model, view, controller, and application Python files can be tedious, repetitive, and error prone when dealing with a large number of widgets. This script auto-generates templates of these files based on an input file containing a list of widgets names. It generates code for the following classes:
- View class
 - imports and builds UI code auto-generated by `pyuic4` or `pyside-uic`
 - getter/setter style properties for the value each widget
 - getter/setter style properties for the enabled state value of each widget
 - `setModel` functions for widget types compatible with Qt models
 - widget signal connections
 - signal event functions
 - a function to update widget values upon announced changes to the model data
- Controller class
 - signal event functions
- Model class
 - getter/setter style properties for the value of each Qt model
 - ConfigParser section and options variables
 - creation of Qt models for compatible widget types
 - variable name placeholders
 - functions for subscribing to and announcing changes to the model data
- Application class
 - instantiation of model, view, and controller classes

See also this Stack Overflow post: http://stackoverflow.com/a/26699122/1470749

Usage
-----

`py_qt_mvc.py widget_names.txt`

where `widget_names.txt` is a text file containing a widget name on each line, and each widget name starts with the widget type, e.g. `pushButton_my_button`.

Example
-------

Given a text file containing the widget name of `comboBox_test`, the following code would be generated:

```python
#####################
# views\MainView.py #
#####################
from PySide import QtGui
from gen.ui_MainView import Ui_MainView

class MainView(QtGui.QMainWindow):

    #### properties for widget value ####
    @property
    def test(self):
        return self.ui.comboBox_test.currentIndex()
    @test.setter
    def test(self, value):
        self.ui.comboBox_test.setCurrentIndex(value)

    #### properties for widget enabled state ####
    @property
    def test_enabled(self):
        return self.ui.comboBox_test.isEnabled()
    @test_enabled.setter
    def test_enabled(self, value):
        self.ui.comboBox_test.setEnabled(value)

    def __init__(self, model, main_ctrl):
        self.model = model
        self.main_ctrl = main_ctrl
        super(MainView, self).__init__()
        self.build_ui()
        # register func with model for model update announcements
        self.model.subscribe_update_func(self.update_ui_from_model)

    def build_ui(self):
        self.ui = Ui_MainView()
        self.ui.setupUi(self)

        #### set Qt model for compatible widget types ####
        self.ui.comboBox_test.setModel(self.model.test_model)

        #### connect widget signals to event functions ####
        self.ui.comboBox_test.currentIndexChanged.connect(self.on_test)

    def update_ui_from_model(self):
        print 'DEBUG: update_ui_from_model called'
        #### update widget values from model ####
        self.test = self.model.test

    #### widget signal event functions ####
    def on_test(self, index): self.main_ctrl.change_test(index)


###########################
# ctrls\MainController.py #
###########################
from PySide import QtGui

class MainController(object):

    def __init__(self, model):
        self.model = model

    #### widget event functions ####
    def change_test(self, index):
        self.model.test = index
        print 'DEBUG: change_test called with arg value:', index


##################
# model\Model.py #
##################
from PySide import QtGui

class Model(object):

    #### properties for value of Qt model contents ####
    @property
    def test_items(self):
        return self.test_model.stringList()
    @test_items.setter
    def test_items(self, value):
        self.test_model.setStringList(value)

    def __init__(self):
        self._update_funcs = []
        self.config_section = 'settings'
        self.config_options = (
            ('test', 'getint'),
        )

        #### create Qt models for compatible widget types ####
        self.test_model = QtGui.QStringListModel()

        #### model variables ####
        self.test = None

    def subscribe_update_func(self, func):
        if func not in self._update_funcs:
            self._update_funcs.append(func)

    def unsubscribe_update_func(self, func):
        if func in self._update_funcs:
            self._update_funcs.remove(func)

    def announce_update(self):
        for func in self._update_funcs:
            func()


##########
# App.py #
##########
import sys
from PySide import QtGui
from model.Model import Model
from ctrls.MainController import MainController
from views.MainView import MainView

class App(QtGui.QApplication):
    def __init__(self, sys_argv):
        super(App, self).__init__(sys_argv)
        self.model = Model()
        self.main_ctrl = MainController(self.model)
        self.main_view = MainView(self.model, self.main_ctrl)
        self.main_view.show()

if __name__ == '__main__':
    app = App(sys.argv)
    sys.exit(app.exec_())


```
