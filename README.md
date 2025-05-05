<!-- Archivo: GarantiaForm.html -->
<!DOCTYPE html>
<html>
<head>
  <base target="_top">
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; }
    label { font-weight: bold; margin-top: 10px; display: block; }
    input, select, textarea { width: 100%; margin-bottom: 10px; padding: 8px; }
    button { padding: 10px 15px; background-color: #4CAF50; color: white; border: none; cursor: pointer; }
    button:hover { background-color: #45a049; }
    .hidden { display: none; }
    #progressBarContainer { margin-top: 10px; display: none; }
    #progressBar { width: 0; height: 20px; background-color: #4CAF50; text-align: center; color: white; line-height: 20px; font-size: 12px; }
    #progressBarBackground { width: 100%; background-color: #ddd; }
  </style>
</head>
<body>
  <div id="login">
    <h2>Login Concesionario</h2>
    <label>Usuario:</label>
    <input type="text" id="usuario">
    <label>Contraseña:</label>
    <input type="password" id="password">
    <button onclick="login()">Ingresar</button>
    <div id="loginError" style="color:red;"></div>
  </div>

  <div id="formulario" class="hidden">
    <h2>Reclamo de Garantía</h2>

    <label>VIN (N° Chasis):</label>
    <input type="text" id="vin">
    <button onclick="validarVIN()">Validar VIN</button>

    <label>Motor:</label>
    <input type="text" id="motor" readonly>
    <label>Modelo:</label>
    <input type="text" id="modelo" readonly>
    <label>Marca:</label>
    <input type="text" id="marca" readonly>
    <label>Kilómetros:</label>
    <input type="number" id="kms">

    <label>Tipo de Reclamo:</label>
    <select id="tipoReclamo">
      <option value="Garantia">Garantía</option>
      <option value="Logistico">Logístico</option>
    </select>

    <label>Descripción de la Falla:</label>
    <textarea id="fallo" rows="4"></textarea>

    <label>Repuesto que ocasionó la falla:</label>
    <input type="text" id="repuestoPrincipal">

    <label>Repuestos reemplazados:</label>
    <textarea id="repuestos" rows="3"></textarea>

    <label>Foto pieza principal (Batch-Code):</label>
    <input type="file" id="fotoPieza" accept="image/*">

    <label>Fotos de repuestos utilizados:</label>
    <input type="file" id="fotoRepuestos" multiple accept="image/*">

    <label>Foto del tablero:</label>
    <input type="file" id="fotoTablero" accept="image/*">

    <button onclick="enviarReclamo()">Registrar Reclamo</button>

    <!-- Barra de carga -->
    <div id="progressBarContainer">
      <div id="progressBarBackground">
        <div id="progressBar">0%</div>
      </div>
    </div>
  </div>

  <script>
    let usuarioActual = '';
    let razonSocial = ''; // Nueva variable

    function login() {
      const usuario = document.getElementById("usuario").value;
      const password = document.getElementById("password").value;

      google.script.run.withSuccessHandler(function(response) {
        if (response.error) {
          document.getElementById("loginError").textContent = response.error;
        } else {
          usuarioActual = usuario;
          razonSocial = response.razon; // Guardar razón social
          document.getElementById("login").classList.add("hidden");
          document.getElementById("formulario").classList.remove("hidden");
        }
      }).loginUsuario(usuario, password);
    }

    function validarVIN() {
      const vin = document.getElementById("vin").value;
      google.script.run.withSuccessHandler(function(data) {
        if (data.error) {
          alert(data.error);
        } else {
          document.getElementById("motor").value = data.motor;
          document.getElementById("modelo").value = data.modelo;
          document.getElementById("marca").value = data.marca;
        }
      }).validarVIN(vin);
    }

    function actualizarBarraProgreso(porcentaje) {
      const progressBar = document.getElementById("progressBar");
      const progressBarContainer = document.getElementById("progressBarContainer");

      progressBarContainer.style.display = "block";
      progressBar.style.width = `${porcentaje}%`;
      progressBar.textContent = `${porcentaje}%`;

      if (porcentaje === 100) {
        setTimeout(() => {
          progressBarContainer.style.display = "none";
          progressBar.style.width = "0%";
          progressBar.textContent = "0%";
        }, 2000);
      }
    }

    function enviarReclamo() {
      const vin = document.getElementById("vin").value;
      const motor = document.getElementById("motor").value;
      const modelo = document.getElementById("modelo").value;
      const marca = document.getElementById("marca").value;
      const kms = document.getElementById("kms").value;
      const tipoReclamo = document.getElementById("tipoReclamo").value;
      const fallo = document.getElementById("fallo").value;
      const repuestoPrincipal = document.getElementById("repuestoPrincipal").value;
      const repuestos = document.getElementById("repuestos").value;

      const pieza = document.getElementById("fotoPieza").files[0];
      const repuestoFotos = document.getElementById("fotoRepuestos").files;
      const tablero = document.getElementById("fotoTablero").files[0];

      actualizarBarraProgreso(10); // Inicia la barra al 10%

      function getBase64(file, callback) {
        const reader = new FileReader();
        reader.onload = () => callback(reader.result);
        reader.readAsDataURL(file);
      }

      let urls = { pieza: '', repuestos: [], tablero: '' };
      let total = 1 + repuestoFotos.length + 1;
      let completados = 0;

      function checkCompleto() {
        completados++;
        actualizarBarraProgreso((completados / total) * 100);

        if (completados === total) {
          google.script.run.withSuccessHandler(function(res) {
            if (res.error) {
              actualizarBarraProgreso(0); // Reinicia la barra en caso de error
              alert(res.message); // Muestra el mensaje en un pop-up
              document.getElementById("kms").focus();
            } else {
              alert(res.message); // Muestra el mensaje en un pop-up
              limpiarFormulario();
              actualizarBarraProgreso(100); // Completa la barra
            }
          }).registrarReclamo({
            usuario: razonSocial,
            vin, motor, modelo, marca, kms, tipoReclamo,
            fallo, repuestoPrincipal, repuestos,
            urlFotoPieza: urls.pieza,
            urlsFotoRepuestos: urls.repuestos.join(', '),
            urlFotoTablero: urls.tablero
          });
        }
      }

      if (pieza) {
        getBase64(pieza, function(data) {
          google.script.run.withSuccessHandler(function(url) {
            urls.pieza = url;
            checkCompleto();
          }).subirArchivo(data, pieza.name, 'pieza');
        });
      } else checkCompleto();

      if (tablero) {
        getBase64(tablero, function(data) {
          google.script.run.withSuccessHandler(function(url) {
            urls.tablero = url;
            checkCompleto();
          }).subirArchivo(data, tablero.name, 'tablero');
        });
      } else checkCompleto();

      if (repuestoFotos.length > 0) {
        Array.from(repuestoFotos).forEach(file => {
          getBase64(file, function(data) {
            google.script.run.withSuccessHandler(function(url) {
              urls.repuestos.push(url);
              checkCompleto();
            }).subirArchivo(data, file.name, 'repuestos');
          });
        });
      } else checkCompleto();
    }

    function limpiarFormulario() {
      document.getElementById("vin").value = "";
      document.getElementById("motor").value = "";
      document.getElementById("modelo").value = "";
      document.getElementById("marca").value = "";
      document.getElementById("kms").value = "";
      document.getElementById("tipoReclamo").value = "Garantia"; // Resetear a valor por defecto
      document.getElementById("fallo").value = "";
      document.getElementById("repuestoPrincipal").value = "";
      document.getElementById("repuestos").value = "";
      document.getElementById("fotoPieza").value = "";
      document.getElementById("fotoRepuestos").value = "";
      document.getElementById("fotoTablero").value = "";
    }
  </script>
</body>
</html>
