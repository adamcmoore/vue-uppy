# Uppy Vue Component
A Vue component for Uppy uploads, supporting single & multiple uploads, companion, and removing 
existing files. This is not an npm package and more of a demonstration for solving the issues discussed here: https://github.com/transloadit/uppy/issues/1112


## Installation
This is in very early development so not yet released as an npm package. To install:

1. Copy uppy.vue to your project.
2. Add `import Uppy from Uppy` in the component you need Uppy in.


## Example Usage
``` html
<uppy
	endpoint="/image-upload-url"
	v-bind:existingFiles="{{ [{id: 1, name: 'test.jpg'}, {id: 2, name: 'test.png'}] }}"
	v-bind:maxFileSizeMb="{{ 4 }}"
	v-bind:allowedFileTypes="{{ ['image/png', 'image/jpg'] }}"
	v-bind:multipleUploads="true"
	placeholderImage="placeholder.png">
</uppy>
```

The `existingFiles` property should be a collection of files already existing in the database.


## Future Development
[ ] Abstract more of the component functionality as component properties. The template currently has the form field names fixed.
[ ] Allow overriding the Uppy options, including the providers.
[ ] Add tests.
[ ] Release as an npm package.

