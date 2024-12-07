<script setup>
import { reactive } from "vue";
import * as Yup from "yup";

const data = reactive({
  name: "",
  email: "",
  ticket_files: [
    {
      file_type: "",
      file: null,
    },
  ],
});

const yup_validation_errors = reactive({});

// get safe number by given any number or string
const safeNumber = (number) => {
  number = parseFloat(number);
  return isNaN(number) ? 0 : number;
};

const yupValidationSchema = () => {
  return Yup.object({
    name: Yup.string().required("Name is required."),
    email: Yup.string().required("Email is required.").email("Email is invalid."),
    message: Yup.string().required("Message is required.").max(500, "Message must not be greater than 500 charaters."),
    ticket_files: Yup.array().of(
      Yup.object({
        file_type: Yup.string()
          .nullable()
          .test(
            "fileTypeRequired",
            "File type is required when file is present.",
            function (value) {
              const { file } = this.parent;
              return file ? value.length : true;
            }
          ),
        file: Yup.mixed()
          .nullable()
          .test(
            "fileRequired",
            "File is required when file type is present.",
            function (value) {
              const { file_type } = this.parent;
              return file_type ? value != null : true;
            }
          )
          .test(
            "fileTypeFile",
            "File must be pdf format.",
            function (value) {
              const { file_type } = this.parent;

              if (value && file_type == "File") {
                if (value instanceof File) {
                  return ["application/pdf"].includes(value?.type);
                } else {
                  return value.endsWith(".pdf");
                }
              } else {
                return true;
              }
            }
          )
          .test(
            "fileTypeImage",
            "File must be jpg, jpeg, png, gif format.",
            function (value) {
              const { file_type } = this.parent;
              if (value && file_type == "Image") {
                if (value instanceof File) {
                  return [
                    "image/jpg",
                    "image/jpeg",
                    "image/png",
                    "image/gif",
                  ].includes(value?.type);
                } else {
                  return !value.endsWith(".pdf");
                }
              } else {
                return true;
              }
            }
          )
          .test(
            "fileSize",
            "File must not be greater than 100KB.",
            (value) =>
              value && value instanceof File
                ? value?.size <= 1024 * 100
                : true
          ),
      })
    ),
  });
};

const validateYupSchema = (field) => {
  yupValidationSchema()
    .validateAt(field, data)
    .then(() => {
      delete yup_validation_errors[field];
    })
    .catch((err) => {
      yup_validation_errors[field] = err.message;
    });
};

const attachNewTicketFile = (event, ticketFileIndex) => {
  data.ticket_files[ticketFileIndex].file =
    event.target.files[0] ?? null;

  validateYupSchema(`ticket_files[${ticketFileIndex}].file`);
  validateYupSchema(`ticket_files[${ticketFileIndex}].file_type`);
};

const addTicketFileRow = () => {
  data.ticket_files.push({
    file_type: "",
    file: null,
  });
};

const deleteTicketFileRow = (ticketFileIndex) => {
  delete data.ticket_files[ticketFileIndex];
};

const showTicketFileAddButton = (ticketFileIndex) => {
  const exactLastIndex = safeNumber(
    Object.keys(data.ticket_files).pop()
  );
  return exactLastIndex === ticketFileIndex;
};

const showTicketFileDeleteButton = () => {
  return Object.keys(data.ticket_files).length > 1;
};

const previewTicketFile = (file) => {
  let fileUrl;
  if (file instanceof File) {
    fileUrl = URL.createObjectURL(file);
    window.open(fileUrl, "_blank");
  }
  // when you're in edit page, file can be an url
  // else if (typeof file === "string") {
  //   fileUrl = file;
  //   window.open(fileUrl, "_blank");
  // }
};

const submit = () => {
  yupValidationSchema()
    .validate(data, { abortEarly: false })
    .then(() => {
      Object.keys(yup_validation_errors).forEach((key) => delete yup_validation_errors[key]);
      // store ticket information
      alert("Ticket created successfully!");
    })
    .catch((err) => {
      Object.keys(yup_validation_errors).forEach((key) => delete yup_validation_errors[key]);
      err.inner.forEach((error) => {
        yup_validation_errors[error.path] = error.message;
      });

      const totalError = Object.keys(yup_validation_errors).length;

      if (totalError > 0) {
        alert(`You need to fill ${totalError} more empty mandatory fields`);
        return false;
      }
    });
};
</script>

<template>
  <section class="bg-gray-50 dark:bg-gray-900">
    <div class="py-8 px-4 mx-auto max-w-screen-xl lg:py-16 grid lg:grid-cols-1 gap-8 lg:gap-16">
      <div>
        <div class="w-full p-6 space-y-8 sm:p-8 bg-white rounded-lg shadow-xl dark:bg-gray-800">
          <h1
            class="mb-4 text-4xl font-extrabold tracking-tight leading-none text-gray-900 md:text-5xl lg:text-6xl dark:text-white">
            Open a new ticket
          </h1>
          <p class="mb-6 text-lg font-normal text-gray-500 lg:text-xl dark:text-gray-400">
            Please, describe your problem. We're here to assist you!
          </p>
          <form class="mt-8 space-y-6" @submit.prevent="submit">
            <div class="grid lg:grid-cols-2 gap-4">
              <div>
                <label for="name" class="block mb-2 text-sm font-medium text-gray-900 dark:text-white">Name</label>
                <input type="text" name="name" id="name"
                  class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
                  v-model="data.name" @input="validateYupSchema('name')" @blur="validateYupSchema('name')" />
                <div v-if="yup_validation_errors.name" class="text-red-800">
                  {{ yup_validation_errors.name }}
                </div>
              </div>

              <div>
                <label for="email" class="block mb-2 text-sm font-medium text-gray-900 dark:text-white">Email</label>
                <input type="text" name="email" id="email"
                  class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
                  v-model="data.email" @input="validateYupSchema('email')" @blur="validateYupSchema('email')" />
                <div v-if="yup_validation_errors.email" class="text-red-800">
                  {{ yup_validation_errors.email }}
                </div>
              </div>
            </div>

            <div>
              <label for="message" class="block mb-2 text-sm font-medium text-gray-900 dark:text-white">Message</label>
              <textarea id="message" rows="4"
                class="block p-2.5 w-full text-sm text-gray-900 bg-gray-50 rounded-lg border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
                placeholder="Write your problem here..." v-model="data.message" @input="validateYupSchema('message')"
                @blur="validateYupSchema('message')"></textarea>
              <div v-if="yup_validation_errors.message" class="text-red-800">
                {{ yup_validation_errors.message }}
              </div>
            </div>

            <div>
              <h1 class="mb-4 text-lg font-bold tracking-tight leading-none text-gray-900">
                Attachments
              </h1>

              <template v-for="(ticketFile, ticketFileIndex) in data.ticket_files"
                :key="`ticket_file_${ticketFileIndex}`">
                <div v-if="ticketFile" class="mb-4">
                  <div class="grid lg:grid-cols-4 gap-4">
                    <div class="col-span-3 grid lg:grid-cols-2 gap-4">
                      <div>
                        <select v-model="ticketFile.file_type"
                          class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
                          @change="() => {
                            validateYupSchema(
                              `ticket_files[${ticketFileIndex}].file_type`
                            );
                            validateYupSchema(
                              `ticket_files[${ticketFileIndex}].file`
                            );
                          }
                            ">
                          <option value="">--Select File Type--</option>
                          <option value="Image">Image</option>
                          <option value="File">File</option>
                        </select>
                        <div v-if="
                          yup_validation_errors[
                          `ticket_files[${ticketFileIndex}].file_type`
                          ]
                        " class="text-red-800">
                          {{
                            yup_validation_errors[
                            `ticket_files[${ticketFileIndex}].file_type`
                            ]
                          }}
                        </div>
                      </div>

                      <div>
                        <input type="file" @change="attachNewTicketFile($event, ticketFileIndex)"
                          class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500" />
                        <div v-if="
                          yup_validation_errors[
                          `ticket_files[${ticketFileIndex}].file`
                          ]
                        " class="text-red-800">
                          {{
                            yup_validation_errors[
                            `ticket_files[${ticketFileIndex}].file`
                            ]
                          }}
                        </div>
                      </div>
                    </div>

                    <div>
                      <button type="button"
                        class="text-white bg-blue-700 hover:bg-blue-800 focus:ring-4 focus:ring-blue-300 font-medium rounded-lg text-sm px-5 py-2.5 me-2 mb-2 dark:bg-blue-600 dark:hover:bg-blue-700 focus:outline-none dark:focus:ring-blue-800"
                        v-if="ticketFile.file" @click="previewTicketFile(ticketFile.file)">
                        View
                      </button>

                      <button type="button"
                        class="focus:outline-none text-white bg-red-700 hover:bg-red-800 focus:ring-4 focus:ring-red-300 font-medium rounded-lg text-sm px-5 py-2.5 me-2 mb-2 dark:bg-red-600 dark:hover:bg-red-700 dark:focus:ring-red-900"
                        v-if="showTicketFileDeleteButton()" @click="deleteTicketFileRow(ticketFileIndex)">
                        Delete
                      </button>

                      <button type="button"
                        class="focus:outline-none text-white bg-green-700 hover:bg-green-800 focus:ring-4 focus:ring-green-300 font-medium rounded-lg text-sm px-5 py-2.5 me-2 mb-2 dark:bg-green-600 dark:hover:bg-green-700 dark:focus:ring-green-800"
                        v-if="showTicketFileAddButton(ticketFileIndex)" @click="addTicketFileRow">
                        Add
                      </button>
                    </div>
                  </div>
                </div>
              </template>
            </div>

            <button type="submit"
              class="w-full px-5 py-3 text-base font-medium text-center text-white bg-blue-700 rounded-lg hover:bg-blue-800 focus:ring-4 focus:ring-blue-300 sm:w-auto dark:bg-blue-600 dark:hover:bg-blue-700 dark:focus:ring-blue-800">
              Submit
            </button>
          </form>
        </div>
      </div>
    </div>
  </section>
</template>
