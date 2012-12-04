# Fault in Android DatePicker

This is a real world example of a __borderline__ fault that occurred in [Android 4.2](https://code.google.com/p/android/issues/detail?id=39692).

This is not a pure integration fault - this is a borderline of unit and integration fault.
The problem mostly belongs to one class `NumberPicker`, it is exposed by the interaction of two classes `DatePicker` and `NumberPicker`.
  
### Brief description  
  
Missing month December when adding new event in the calendar from contact list.

![ScreenShot](https://android.googlecode.com/issues/attachment?aid=396920000000&name=Screenshot_2012-11-15-12-42-36.png&token=0dwjif79fp2StvUekdZzPsOXfQM%3A1354615411947&inline=1)

## The meat  

[`android.widget.NumberPicker`](https://developer.android.com/guide/topics/ui/controls/pickers.html) is a widget that enables the user to select a number form a predefined range. It is used by many range pickers, including `TimePicker` (for selecting the time of day, in either 24 hour or AM/PM mode) and a customizable `DatePicker` where the minimal and maximal date from which dates to be selected can be customized.
  
### Failure  
  
`DatePicker` uses `NumberPicker` to set `maxValue` to 12 for the string array "months" that is 12 entries long.

```java
            mMonthPicker.setMinValue(1);
            mMonthPicker.setMaxValue(12);
            mMonthPicker.setDisplayedValues(months);
```

However, the values change as soon as the display values are set via `setDisplayedValues(String[])`. `maxValue` gets truncated to 11 which in the end skips "December" as it is the 12th and last month.

### Root cause  

The documentation for `setMinValue()` and `setMaxValue()` says: "Sets the min value of the picker." and "Sets the max value of the picker." respectively.

`maxValue` gets truncated to 11, because in the method `setDisplayedValues(String[])` the values in the range represent the indices of the array:

```java
            if (getMaxValue() >= displayedValues.length) {
                setMaxValue(displayedValues.length - 1);
            }
```

To summarize: the problem comes from the lack of documentation about the relations between the methods of a class `NumberPicker`: `setMinValue()`, `setMaxValue()` and `setDisplayedValues(String[])`.
                                                              
## Source code 

`DatePicker`'s code stays as is:    

```java
        /* ... */
        mMonthPicker = (NumberPicker) findViewById(R.id.month);
        mMonthPicker.setFormatter(NumberPicker.getTwoDigitFormatter());
        DateFormatSymbols dfs = new DateFormatSymbols();
        String[] months = dfs.getShortMonths();

        /*
         * If the user is in a locale where the month names are numeric,
         * use just the number instead of the "month" character for
         * consistency with the other fields.
         */
        if (months[0].startsWith("1")) {
            for (int i = 0; i < months.length; i++) {
                months[i] = String.valueOf(i + 1);
            }
            mMonthPicker.setMinValue(1);
            mMonthPicker.setMaxValue(12);
        } else {
            mMonthPicker.setMinValue(1);
            mMonthPicker.setMaxValue(12);
            mMonthPicker.setDisplayedValues(months);
        }
        /* ... */
```        

`NumberPicker`'s methods before [fix](https://github.com/android/platform_frameworks_base/commit/7018cfdc05dc6135949806749ff5c370dce09ced):

```java
    public void setDisplayedValues(String[] displayedValues) {
        if (mDisplayedValues == displayedValues) {
            return;
        }
        mDisplayedValues = displayedValues;
        if (mDisplayedValues != null) {
            // Allow text entry rather than strictly numeric entry.
            mInputText.setRawInputType(InputType.TYPE_CLASS_TEXT
                    | InputType.TYPE_TEXT_FLAG_NO_SUGGESTIONS);
            // Make sure the min, max, respect the size of the displayed
            // values. This will take care of the current value as well.
            if (getMinValue() >= displayedValues.length) {
                setMinValue(0);
            }
            if (getMaxValue() >= displayedValues.length) {
                setMaxValue(displayedValues.length - 1);
            }
        } else {
            mInputText.setRawInputType(InputType.TYPE_CLASS_NUMBER);
        }
        updateInputTextView();
        initializeSelectorWheelIndices();
        tryComputeMaxWidth();
    }
```

`NumberPicker`'s methods after [fix](https://github.com/android/platform_frameworks_base/commit/7018cfdc05dc6135949806749ff5c370dce09ced):

```java
     /**
     * Sets the values to be displayed.
     *
     * @param displayedValues The displayed values.
     *
     * <strong>Note:</strong> The length of the displayed values array
     * must be equal to the range of selectable numbers which is equal to
     * {@link #getMaxValue()} - {@link #getMinValue()} + 1.
     */
    public void setDisplayedValues(String[] displayedValues) {
        if (mDisplayedValues == displayedValues) {
            return;
        }
        mDisplayedValues = displayedValues;
        if (mDisplayedValues != null) {
            // Allow text entry rather than strictly numeric entry.
            mInputText.setRawInputType(InputType.TYPE_CLASS_TEXT
                    | InputType.TYPE_TEXT_FLAG_NO_SUGGESTIONS);
        } else {
            mInputText.setRawInputType(InputType.TYPE_CLASS_NUMBER);
        }
        updateInputTextView();
        initializeSelectorWheelIndices();
        tryComputeMaxWidth();
    }  

```



---
[back to Wiki](https://github.com/rubinovk/integration-faults/wiki)
