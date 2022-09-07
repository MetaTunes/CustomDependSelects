# CustomDependSelects

**TLDR:** Use custom selectors in page field selector: 

- check_access=role1|role2 ... - to control who can see results
- field=[item.id] - to select on id of repeater item containing the page field.

## Summary

This module extends the capabilities of selectors specified in the 'input' tab of a page reference field, specifically when that field is part of a set of dependent selectors which may be inside a repeater item. This readme also attempts to bring together various existing documentation regarding the use of dependent page selectors and the enhancements that were provided by various PW versions.

## Background

Dependent selectors are where one selector field (the 'secondary' or dependent selector) has options which depend on another selector field (the 'primary' field). In a simple case, this is achieved by placing (in the input tab selector string) something like

```php
parent=page.primary_page_field
```

This will provide, as the selectable options for the secondary field, all pages having the primary field selection as their parent. Unfortunately this does not work if the fields in question are contained within a repeater (or repeaterMatrix) item. This could be fixed by a hook on InputfieldPage::getselectablePages(), but a better solution was introduced by [@ryan](https://processwire.com/talk/profile/2-ryan/) in PW v 3.0.200 - namely the use of 'item' in place of 'page' (see https://processwire.com/talk/topic/26999-page-field-based-on-parent-in-matrix-or-repeater-field/ for more information).

However, there were still some issues with that solution:

1. The example in https://processwire.com/talk/topic/26999-page-field-based-on-parent-in-matrix-or-repeater-field/?do=findComment&comment=223610 of 'category=item.id' does not work - the field after the '=item.' term needs to be an inputfield in the repeater item, which is not the case with id. Also, category=[item.id] does not work (analagous to [page.id] - see https://processwire.com/docs/selectors/#api-variables-in-selectors), although [page.id] does not seem to work in this context either.
2. The use of the term '.owner.' did not work inside a repeater. This term was introduced in PW v3.0.95 - see https://processwire.com/blog/posts/processwire-3.0.95-core-updates/. One very good use of this is when the secondary page select field is a repeater and you want to reference a field in its 'owner' page, which is, of course, different from its actual parent. Unfortunately, this did not work if the page select fields were inside a repeater. This was fixed in v3.0.203.
3. For anyone other than a superuser, if dependent select fields are repeaters, then the selectors described above do not work unless the user has a role with access specifically granted to the related repeater template. This is because the default access is inherited from the parent (which is 'admin') not the 'owner'. Using 'check_access=0' in the selector does not work because the options list is provided by ProcessPageSearch which strips check_access from the selector items, if the user is not a superuser. See the discussion at https://processwire.com/talk/topic/27533-permit-access-to-repeater-field-pages/. Although this issue can be resolved by setting the repeater template access permissions explicitly, that is somewhat cumbersome and inflexible (the permissions will operate outside the intended selector context which may be undesirable).

This module is designed to fix problems 1 & 3 above. It requires usage of at least v3.0.200 (plus you will need v3.0.203 if you want a fix for problem 2).

## Usage

After installing the module, usage is very simple.

If you want to refer to the id of the current repeater item (i.e. the one holding the page select in question) just use "field=[item.id]". (I may extend this later to include other fields than id, if required.)

If you want to permit access to a number of roles (as well as superuser), such as role1 and role2, just use "check_access=role1|role2".

## Worked example

This example is designed to illustrate all of the above. The context is my new web-based app for cider production and sales management "CiderMaster". Among other things, this has a template - 'CIder' - for recording all stages of production and use. The stages are recorded in a repeaterMatrix field called 'stage'. Two such stages are 'blend_from' and 'blend_to' to record the blending of (part of) one cider to another. The blend_from stage has a page field 'blendFrom' which links to the blend_to stage in the source cider and the 'blend_to' stage has a field 'blendTo' which links back to the blend_from stage in the target cider. The user actions are to create the blend_to stage in the source cider first (unlinked) and then to create the blend_from stage in the target cider and link it to an (unlinked) blend_to stage. Ciders are grouped under 'Year' parents.

To implement this, we have 3 page-select fields (year, cider and blendFrom) which are contained within the blend_from stage (which, remember, is a repeaterMatrix item). The process is as follows (assuming all blend_to stages have already been created):

1. The initial primary page select is 'Year'. Until this is selected, no other selections are possible:

![No_selection](M:\laragon\www\Cider\site\modules\CustomDependSelects\images\No_selection.jpg)

(Usually, I actually hide the dependent fields until the primary has been selected, but in this case you can see them and that no options are available).

2. After selecting a year, cider options are available (but not 'blend from' options):

![Year_selected](M:\laragon\www\Cider\site\modules\CustomDependSelects\images\Year_selected.jpg)

3. Then, after selecting a cider, any unlinked 'blend_to' stages are available in the blendFrom field:

![Cider_selected](M:\laragon\www\Cider\site\modules\CustomDependSelects\images\Cider_selected.jpg)

This is achieved by using the following selectors:

- In the field 'cider' we use this selector: 

  ```php
  hasParent=item.year, template=Cider|Juice
  ```

  (Note that we have Cider and Juice templates which are fairly similar, but Juice is for unfermented cider). This selects all ciders and juices with a parent equal to the currently selected year in the current blend_from item.

- In the field 'blendFrom' we use this selector: 

  ```php
  template=repeater_stage, repeater_matrix_type=2|11, stage.owner.id=item.cider, blendTo=0|[item.id], check_access=webmaster|editor
  ```

  Here we are using 'stage.owner.id' to select based on the cider field we just selected. We can't use 'parent' because the required page is a repeater. 'repeater_matrix=2|11' are blend_to stages (2 for cider, 11 for juice). 'blendTo=0|[item.id]' selects only blend_to stages that have not already been linked elsewhere*. 'check_access=webmaster|editor' will allow users with webmaster or editor roles to see the options, even though the repeater_stage template does not permit them, as it is a descendant of admin and has no explicit access set.

  *Note that if we do not include [item.id], then after saving the page, the current selection will not be shown, because it is not in the list of options (all blend_to stages having been linked - in this case automatically by virtue  of ConnectPageFields, thanks [@Robin S](https://processwire.com/talk/profile/2897-robin-s/)).