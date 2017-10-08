<template>
  <div id="app" class="container">
    <div class="row">
        <div class="col-md-7">
            <table class="table table-striped">
                <tr>
                    <th>Naam</th>
                    <th>Straatnaam</th>
                    <th>Huisnummer</th>
                    <th>Postcode</th>
                    <th>Woonplaats</th>
                    <th>Land</th>
                    <th>&nbsp;</th>
                </tr>
                <tr v-for="address in addresses">
                    <td>
                        <input type="text" class="form-control" v-model="address.title" @change="inputChange" />
                    </td>
                    <td>
                        <input type="text" class="form-control" v-model="address.address" @change="inputChange" />
                    </td>
                    <td>
                        <input type="text" class="form-control" v-model="address.address_number" @change="inputChange" />
                    </td>
                    <td>
                        <input type="text" class="form-control" v-model="address.postal_code" @change="inputChange" />
                    </td>
                    <td>
                        <input type="text" class="form-control" v-model="address.city" @change="inputChange" />
                    </td>
                    <td>
                        <input type="text" class="form-control" v-model="address.country" @change="inputChange" />
                    </td>
                    <td>
                        <button class="btn btn-sm btn-danger" @click="deleteAddress(address)" v-if="isNotEmpty(address)">
                             &#128465;
                        </button>
                    </td>
                </tr>
            </table>
        </div>
        <div class="col-md-5">
            <div class="btn-group special" role="group">
                <button type="button" class="btn btn-sm btn-primary" @click="exportCsv">
                    &#x21D1;
                    Export CSV
                </button>
                <button type="button" class="btn btn-sm btn-primary" @click="uploadFile">
                    &#x21D3;
                    Import CSV
                </button>
                <button type="button" class="btn btn-sm btn-primary" @click="genPdf">
                    &#x21bb;
                    Bijwerken
                </button>
                <input ref="file_input" type="file" style="display:none;" @change="uploadedFile"/>
            </div>
            <div class="row">
                <div class="col-md-12">
                    <iframe width="100%" height="600px" :src="pdf"></iframe>
                </div>
            </div>
        </div>
    </div>

  </div>
</template>

<script>
import PDFDocument from 'pdfkit';
import blobStream from 'blob-stream';
import Papa from 'papaparse';
export default {
  name: 'app',
  data() {
      return {
          addresses: [
                {
                    title: '',
                    address: '',
                    address_number: '',
                    postal_code: '',
                    city: '',
                    country: ''
                }
          ],
          pdf: ''
      }
  },
  mounted: function () {
      if (localStorage.getItem('addresses') !== null) {
          this.addresses = JSON.parse(localStorage.getItem('addresses'));
      }
      this.genPdf();
  },
  methods: {
    genPdf: function () {
        var doc = new PDFDocument({size: 'a4'})
        var stream = doc.pipe(blobStream())

        doc.font('Times-Roman', 14);
        this.paintLines(doc);
        var that = this;
        this.addresses.filter(function (address) {
            return that.isNotEmpty(address);
        }).forEach(function (address, i) {
            doc.text(
                that.presentAddress(address),
                14+((i % 3)*199),
                14+(Math.floor((i % 24) / 3) * 105),
                {
                    width: 171,
                    height: 80
                }
            );

            if (i > 1 && 0 === (i % 23)) {
                doc.addPage();

                that.paintLines(doc);
            }
        });

        doc.end()

        stream.on('finish', function () {
            that.pdf = stream.toBlobURL('application/pdf');
        });
    },
    inputChange: function () {
        if (0 == this.addresses.length || this.isNotEmpty(this.addresses[this.addresses.length - 1])) {
            this.addresses.push({
                title: '',
                address: '',
                address_number: '',
                postal_code: '',
                city: '',
                country: ''
            });
        }

        this.saveToLocalStorage();
    },
    saveToLocalStorage: function () {
        localStorage.setItem('addresses', JSON.stringify(this.addresses));
    },
    isNotEmpty: function(address) {
        return address.title ||
            address.address ||
            address.address_number ||
            address.postal_code ||
            address.city ||
            address.country;
    },
    presentAddress: function (address) {
        return address.title + '\n' +
            address.address+' '+
            address.address_number+
            (address.address_number_addition ? address.address_number_addition : '') + '\n' +
            address.postal_code+' '+address.city +
            (address.country ? ('\n'+address.country) : '');
    },
    paintLines: function (doc) {
        doc
           .moveTo(200, 0)
           .lineTo(200, 841)
           .stroke();
        doc
           .moveTo(400, 0)
           .lineTo(400, 841)
           .stroke();

        for(var i = 1; i < 8; i++) {
            doc
               .moveTo(0, i * 105)
               .lineTo(595, i * 105)
               .stroke();
        }
    },
    deleteAddress: function (address) {
        this.addresses.splice(this.addresses.indexOf(address), 1);
    },
    exportCsv: function () {
        var csvContent = "data:text/csv;charset=utf-8,";
        var that = this;
        csvContent += Papa.unparse(this.addresses.filter(function (address) {
            return that.isNotEmpty(address);
        }), {
            quotes: true
        });
        var encodedUri = encodeURI(csvContent);
        var link = document.createElement("a");
        link.setAttribute("href", encodedUri);
        link.setAttribute("download", "my_data.csv");
        document.body.appendChild(link); // Required for FF

        link.click();
    },
    uploadFile: function () {
        this.$refs.file_input.click();
    },
    uploadedFile: function (e) {
        var files = e.target.files || e.dataTransfer.files;
        if (!files.length) {
            return;
        }

        var that = this;
        Papa.parse(files[0], {
            header: true,
            complete: function (results) {
                that.addresses = results.data;
                that.saveToLocalStorage();
                that.genPdf();
            }
        });
    }
  }
}
</script>

<style lang="scss">
    table {
        font-size: 75%;
    }

    input[type="text"] {
        font-size: 95%;
    }

    table td {
        padding: 0.5rem 0 !important;
    }

    .btn-group.special {
        display: flex;
    }

    .special .btn {
        flex: 1
    }

    .jumbotron {
        padding: 2rem;
    }
</style>
