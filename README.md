# Calendar generation / join 

Calendar generation functions for:
- master calendar (incl. DE/EN date field, fields for shifted fiscal year, month, ...
- n calendar tables (with field prefix)
- join of reduced basis calendar into date source table

The generated calendar fields includes common date parts and formatting style as well as fiscal year and month values.

## Variables needed:
- vCalendarStart
- vCalendarEnd
- vDataModelLanguage = 'EN';

## CALLs
Various call methods are available. The call startCalendarService is necessary.

### CALL startCalendarService()
Setup all descriptive variables
### CALL generateCalendar (vCalendarQualifier) 
Generate three tables:
- calendar
- calendar_time_range
- time_ranges
vCalendarQualifier is optional

### CALL joinCalendar (targetTable, srcDateField, fieldPrefix)
Generate addininal fields in **targetTable** based on date field **srcDateField** with (optional) prefix **fieldPrefix** 

### CALL stopCalendarService;
Drop all tmp tables and fields