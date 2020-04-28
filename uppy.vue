<template>
	<div>
		<div v-for="(file, index) in files">
			<input type="hidden" v-bind:name="'files['+index+'][id]'" :value="file.id">
			<input type="hidden" v-bind:name="'files['+index+'][name]'" :value="file.name">
		</div>

		<div class="upper-wrapper">
			<loading :active.sync="is_loading" :is-full-page="false"></loading>
			<div class="uppy"></div>
		</div>
	</div>
</template>

<script>
const UppyCore        = require('@uppy/core');
const UppyXHRUpload   = require('@uppy/xhr-upload');
const UppyDashboard   = require('@uppy/dashboard');
const UppyStatusBar   = require('@uppy/status-bar');
const UppyInformer    = require('@uppy/informer')
const UppyDragDrop    = require('@uppy/drag-drop');
const UppyDropbox     = require('@uppy/dropbox');
const UppyGoogleDrive = require('@uppy/google-drive');
const UppyOneDrive    = require('@uppy/onedrive');
require('@uppy/core/dist/style.css');
require('@uppy/dashboard/dist/style.css')
require('@uppy/drag-drop/dist/style.css');

export default {
	name: 'uppy',
	components: {},
	props: {
		existingFiles:    { type: Array },   // Collection of objects with id and name properties. Ie [ { id: 1, name: 'test.png', id: 2, name: 'lolcat.jpg'}]
		endpoint: 		  { type: String },  // The URL to send the upload request to
		maxFileSizeMb:    { type: Number },  // max size, in megabytes, for each individual file uploaded
		allowedFileTypes: { type: Array },   // Array of file extensions - eg: ['image/*', '.jpg', '.jpeg', '.png', '.gif']
		multipleUploads:  { type: Boolean }, // Allow uploading multiple images
		disabled:  		  { type: Boolean }, // Disable uplaoding new images
		placeholderImage: { type: String },  // PNG image to display when image doesn't exist on server
		csrfToken:        { type: String }   // CSRF for Companion to send back to the server 
	},
	data () {
		return {
			uppy: null,
			files: [],
			is_loading: true,
		}
	},
	mounted () {
		let _self = this;
		

		if (!_self.placeholderImage) {
			_self.placeholderImage = 'placeholder.png';
		}


		_self.setNotLoading = function() {
			_self.is_loading = false;
			_self.uppy.setOptions({ autoProceed: true });
		}


		_self.initialiseExistingFiles = function() {
			_self.files = _self.existingFiles;

			let num_loading = _self.existingFiles.length;
			if (num_loading === 0) {
				_self.setNotLoading();
			}

			_self.existingFiles.forEach(function(file) {
				// Support unsaved files 
				if (!file.cdn_url) {
					file.cdn_url = _self.placeholderImage;
					file.mimetype = 'image/png';
				}

				window.axios.get(file.cdn_url, { responseType: 'blob'}).then(function(response) {
					_self.uppy.addFile({
						name: file.name,
						type: file.mimetype,
						data: response.data,
						meta: { existing: true },
					 });

					num_loading--;
					if (num_loading === 0) {
						_self.setNotLoading();
					}
				});
			});
		}


		_self.getExistingUppyFiles = function() {
			return _self.uppy.getFiles().filter(function(file) {
				return file.meta.hasOwnProperty('existing');
			});
		}


		_self.setExistingFilesAsUploaded = function() {
			_self.getExistingUppyFiles().forEach(function(file) {
				_self.uppy.setFileState(file.id, { 
					progress: { 
						uploadComplete: true,
						uploadStarted: true
					} 
				});
			});
		}


		_self.setExistingFilesAsNotUploaded = function() {
			_self.getExistingUppyFiles().forEach(function(file) {
				_self.uppy.setFileState(file.id, { 
					progress: { 
						uploadComplete: false,
						uploadStarted: false
					} 
				});
			});			
		}


		_self.initialiseUppy = function() {
			const provider_conf = {
				target: UppyDashboard,
				companionUrl: COMPANION_URL
			};

			_self.uppy = UppyCore({
				multipleUploads: _self.multipleUploads,
				autoProceed: false,
				locale: {
					strings: {
						'cancel': 'Select New File'
					}
				},
				restrictions: {
					maxFileSize: _self.maxFileSizeMb * 1024 * 1024,
					allowedFileTypes: _self.allowedFileTypes,
					maxNumberOfFiles: (_self.multipleUploads ? 100 : 1),
				}}).
				use(UppyDashboard, {
					inline: true,
					target: '.uppy',
					height: 500,
					thumbnailWidth: 280,
					replaceTargetContent: true,
					showProgressDetails: true,
					closeModalOnClickOutside: true,
					hideUploadButton: true,
					hideRetryButton: true,
					hideCancelButton: _self.disabled || _self.multipleUploads,
				}).
				use(UppyXHRUpload, {
					endpoint: _self.endpoint,
					headers: {'X-CSRF-TOKEN': _self.csrfToken}
				}).
				use(UppyDragDrop, {}).
				use(UppyDropbox, provider_conf).
				use(UppyGoogleDrive, provider_conf).
				use(UppyOneDrive, provider_conf);
		}


		_self.initialiseUppy();
		_self.initialiseExistingFiles();


		_self.uppy.on('file-added', function(file) {
			if (!file.meta.hasOwnProperty('existing')) {
				_self.setExistingFilesAsUploaded();
			}
		});


		_self.uppy.on('file-removed', function(file) {
			_self.files = _self.files.filter(function(existing_file) {
				return file.name != existing_file.name;
			});
		});


		_self.uppy.on('upload-success', (file, response) => {
			_self.setExistingFilesAsNotUploaded();

			if (response.body.file) {
				_self.files.push(response.body.file);
			}
		});


		_self.uppy.on('upload-error', (file, error, response) => {
			_self.setExistingFilesAsNotUploaded();

			// For some reason Uppy does not offer a way to remove the single 
			// file after it fails, so we'll do that here.
			_self.uppy.removeFile(file.id);

			if (response.body.hasOwnProperty('files.0')) {
				response.body['files.0'].forEach(function(error) {
					_self.uppy.info(error, 'error', 3000)
				});
			}
		});

	}
}
</script>