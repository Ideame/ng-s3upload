angular.module('ngS3upload.directives', []).
  directive('s3Upload', ['$parse', 'S3Uploader', function ($parse, S3Uploader) {
    return {
      restrict: 'AC',
      require: '?ngModel',
      replace: true,
      transclude: false,
      scope: true,
      controller: ['$scope', '$element', '$attrs', '$transclude', function ($scope, $element, $attrs, $transclude) {
        $scope.attempt = false;
        $scope.success = false;
        $scope.uploading = false;

        $scope.barClass = function () {
          return {
            "bar-success": $scope.attempt && !$scope.uploading && $scope.success
          };
        };
      }],
      compile: function (element, attr, linker) {
        return {
          pre: function ($scope, $element, $attr) {
            if (angular.isUndefined($attr.bucket)) {
              throw Error('bucket is a mandatory attribute');
            }
          },
          post: function (scope, element, attrs, ngModel) {
            // Build the opts array
            var opts = angular.extend({}, scope.$eval(attrs.s3UploadOptions || attrs.options));
            opts = angular.extend({
              submitOnChange: true,
              getOptionsUri: '/getS3Options',
              acl: 'public-read',
              uploadingKey: 'uploading',
              folder: '',
              enableValidation: true
            }, opts);
            var bucket = scope.$eval(attrs.bucket);

            scope.chooseFileLabel = attrs.chooseFileLabel || 'Choose file';
            scope.replaceFileLabel = attrs.replaceFileLabel || 'Replace file';
            scope.storedFileLabel = attrs.storedFileLabel || 'Stored File';
            scope.buttonClass = attr.buttonClass || 'btn btn-primary';

            // Bind the button click event
            var button = angular.element(element.children()[0]),
              file = angular.element(element.find("input")[0]);
            button.bind('click', function (e) {
              file[0].click();
            });

            // Update the scope with the view value
            ngModel.$render = function () {
              scope.filename = ngModel.$viewValue;
            };

            var uploadFile = function () {
              var selectedFile = file[0].files[0];
              var filename = selectedFile.name;
              var ext = filename.split('.').pop();

              scope.$apply(function () {
                S3Uploader.getUploadOptions(opts.getOptionsUri).then(function (s3Options) {
                  if (opts.enableValidation) {
                    ngModel.$setValidity('uploading', false);
                  }

                  var s3Uri = 'https://' + bucket + '.s3.amazonaws.com/';
                  S3Uploader.upload(scope,
                      s3Uri,
                      s3Options.fileKey,
                      s3Options.acl,
                      s3Options.contentType,
                      s3Options.s3Key,
                      s3Options.s3PolicyBase64,
                      s3Options.s3Signature,
                      selectedFile
                    ).then(function () {
                      ngModel.$setViewValue(s3Options.fileUrl);
                      scope.filename = ngModel.$viewValue;

                      if (opts.enableValidation) {
                        ngModel.$setValidity('uploading', true);
                        ngModel.$setValidity('succeeded', true);
                      }
                    }, function () {
                      scope.filename = ngModel.$viewValue;

                      if (opts.enableValidation) {
                        ngModel.$setValidity('uploading', true);
                        ngModel.$setValidity('succeeded', false);
                      }
                    });

                }, function (error) {
                  throw Error("Can't receive the needed options for S3 " + error);
                });
              });
            };

            element.bind('change', function (nVal) {
              if (opts.submitOnChange) {
                uploadFile();
              }
            });
          }
        };
      },
      template: '<div class="upload-wrap">' +
        '<button class="{{ buttonClass }}" type="button"><span ng-if="!filename">{{ chooseFileLabel }}</span><span ng-if="filename">{{ replaceFileLabel }}</span></button>' +
        '<a ng-href="{{ filename  }}" target="_blank" class="" ng-if="filename" >{{ storedFileLabel }}</a>' +
        '<div class="progress progress-striped" ng-class="{active: uploading}" ng-show="attempt" style="margin-top: 10px">' +
        '<div class="bar" style="width: {{ progress }}%;" ng-class="barClass()"></div>' +
        '</div>' +
        '<input type="file" style="display: none"/>' +
        '</div>'
    };
  }]);