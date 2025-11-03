# 1. Creación del proyecto

Se crea el proyecto como spring starter project, con las siguientes dependecnias

- Spring Web
- Spring for Apache ActiveMQ Artemis
![[Pasted image 20251102152533.png]]
# 2. Estructura del proyecto

Creamos los siguientes paquetes dentro del principal

* \*.business
* \*.dto
* \*.jms
* * \*.common

![[Pasted image 20251102153442.png]]

# 3. JMS message:

copiamos esto en un archivo dentro del paquete jms:
`JmsSender.java`
```java

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.stereotype.Service;

import com.example.dto.JmsMessage;

/**
 * 
 */
@Service
public class JmsSender {
	/**
	 * Plantilla JMS para enviar mensajes proporcionada 
	 * e instanciada por Spring.
	 */
	@Autowired
	private JmsTemplate jmsTemplate;

	/**
	 * Envía un mensaje JMS a la cola especificada en el mensaje.
	 * 
	 * @param outMsg Información del mensaje JMS a enviar.
	 */
	public void send(JmsMessage outMsg) {
		jmsTemplate.send(outMsg.getOutQueue(), session -> {
			// Crea un mensaje de texto JMS con el contenido del mensaje.
			jakarta.jms.TextMessage txtMessage = session.createTextMessage(outMsg.getMessage());
			// Añade las propiedades al mensaje JMS.
			if (outMsg.getProperties() != null) {
				outMsg.getProperties().forEach((key, value) -> {
					try {
						txtMessage.setStringProperty(key, value);
					} catch (jakarta.jms.JMSException e) {
						e.printStackTrace();
					}
				});
			}
			return txtMessage;
		});
		System.out.println("Sent message: " + outMsg.getMessage() + " to queue " + outMsg.getOutQueue());
	}
}

```

El siguiente en el paquete dto
`JmsMessage.java`
```java

import java.util.Enumeration;
import java.util.HashMap;
import java.util.Map;

import jakarta.jms.JMSException;
import jakarta.jms.Message;
import jakarta.jms.TextMessage;

import com.example.common.AppException;

/**
 * Clase reprensenta un mensaje JMS.
 */
public class JmsMessage {

	/** Contiene el texto del mensaje JMS. */
	private String message;

	/**
	 * Contiene las propiedades del mensaje JMS.
	 */
	private Map<String, String> properties;

	/** Cola de salida. */
	private String outQueue;

	/**
	 * Constructor.
	 * 
	 * @param message
	 * @throws JMSException
	 */
	@SuppressWarnings("rawtypes")
	public JmsMessage(Message message) {
		TextMessage textMessage = (TextMessage) message;
		try {
			this.message = textMessage.getText();
			// Inicializa el mapa de propiedades.
			properties = new HashMap<>();
			Enumeration propertyNames = textMessage.getPropertyNames();
			// Por cada propiedad del mensaje JMS, la añade al mapa.
			while (propertyNames.hasMoreElements()) {
				// Obtiene el nombre de la propiedad.
				String propertyName = (String) propertyNames.nextElement();
				if (propertyName.startsWith("JMS")) {
					// Ignora las propiedades JMS.
					continue;
				}
				// Añade la propiedad al mapa.
				properties.put(propertyName, textMessage.getStringProperty(propertyName));
			}
		} catch (JMSException e) {
			e.printStackTrace();
			throw new AppException(e);
		}
	}

	/**
	 * Constructor.
	 * 
	 * @param message  El texto del mensaje.
	 * @param headers  Las propiedades del mensaje.
	 * @param queueOut La cola de salida.
	 */
	public JmsMessage(String message, Map<String, String> headers, String queueOut) {
		this.message = message;
		this.properties = headers;
		this.outQueue = queueOut;
	}

	/**
	 * @return the message
	 */
	public final String getMessage() {
		return message;
	}

	/**
	 * @param message the message to set
	 */
	public final void setMessage(String message) {
		this.message = message;
	}

	/**
	 * @return the properties
	 */
	public final Map<String, String> getProperties() {
		return properties;
	}

	/**
	 * @param properties the properties to set
	 */
	public final void setProperties(Map<String, String> properties) {
		this.properties = properties;
	}
	
	/**
	 * Representación String del objeto.
	 */
	public String toString() {
		return "JmsMessage [message=" + message + ", properties=" + properties + "]";
	}

	/**
	 * @return the outQueue
	 */
	public final String getOutQueue() {
		return outQueue;
	}

	/**
	 * @param outQueue the outQueue to set
	 */
	public final void setOutQueue(String outQueue) {
		this.outQueue = outQueue;
	}
}


```

Y este en common

`AppException.java`
```java
  

/**
* Copyright (c) 2025.
*/
  
/**
* Excepción personalizada not-checked:runtime exception:silenciosa.
*/

public class AppException extends RuntimeException {  

/** Serial version UID. */
private static final long serialVersionUID = 1L;

	/**
	* Constructor que recibe un mensaje.
	* @param message Mensaje de la excepción.
	*/
	public AppException(String message) {
		super(message);
	}
	
	/**
	* Constructor que recibe una excepción.
	* @param e Excepción original.
	*/
	public AppException(Exception e) {
		super(e);
	}
}
```

# 4. Confguración Con Artemis

Modificamos el application.propierties y añadimos la configuracion de artemis:

```sh
spring.activemq.broker-url=tcp://localhost:61616
spring.artemis.user=admin
spring.artemis.password=admin
```

> Mi instancia de Artemis fue hecha con docker, el comando es el siguiente por si lo quieres usar:
> `docker run -d --name artemisbroker -e ARTEMIS_USER=admin -e ARTEMIS_PASSWORD=admin -p 8161:8161 -p 61616:61616 apache/activemq-artemis:latest`


# 5. Construccion del dto:

La definición del mensaje es la siguiente, tanto del requets como el response 
```json
// Request:
{
	"cantidades[TU_NOMBRE]": [
		10,
		20,
		30
	]
}
// Response:
{
	"total[TU_APELLIDO_PATERNO]": 60
}
```

Creamos el dto de cantidades\[tunombre\], se usa una lista de enteros porque es lo que es cantidades\[tunombre\]

`cantidadesRequest.java`
```java
package com.example.dto;

import java.util.List;

public class cantidadesRequest {
	private List<Integer> cantidadesArturo;

	public List<Integer> getCantidadesArturo() {
		return cantidadesArturo;
	}
	public void setCantidadesArturo(List<Integer> cantidadesArturo) {
		this.cantidadesArturo = cantidadesArturo;
	}

}
```

para cantidadesResponse:

`cantidadesResponse.java`
```java
package com.example.dto;

public class cantidadesResponse {
	private int totalGarcia;
	
	public int getTotalGarcia() {
		return totalGarcia;
	}

	public void setTotalGarcia(int totalGarcia) {
		this.totalGarcia = totalGarcia;
	}
}
```

y construimos el servicio que sumara todos los datos de la lista de integer:

`calcular.java`
```java
package com.example.business;

import org.springframework.stereotype.Service;

import com.example.dto.cantidadesRequest;

@Service
public class calcular {
	
	public int calcularByRequest(cantidadesRequest request) {
		return request.getCantidadesArturo().stream() 
				.mapToInt(Integer::intValue).sum(); // forma de tratar la lista en un stream!
	}

}
```

# 6. Consumir de la cola y enviarlo:

Creamos la clase consumidora en el paquete jms:

`CalcularConsumer.java`
```java
package com.example.jms;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jms.annotation.JmsListener;
import org.springframework.stereotype.Component;


import jakarta.jms.Message;

import com.example.business.calcular;
import com.example.dto.JmsMessage;
import com.example.dto.cantidadesRequest;
import com.example.dto.cantidadesResponse;
import com.google.gson.Gson;

@Component
public class CalcularConsumer {
	@Autowired
    private JmsSender jmsSender;
	
	@Autowired
	private calcular cal;
	
	@JmsListener(destination = "calcular.in")
	public void readMessageFromInput(Message msg) {
		JmsMessage jmsMessage = new JmsMessage(msg);
		// El json que parsea json a clases
		Gson json = new Gson();
		// Convertimos el json a una clase request
		cantidadesRequest request = json.fromJson(jmsMessage.getMessage(), cantidadesRequest.class);
		
		// Prepara la respuesta
		cantidadesResponse response = new cantidadesResponse();
		response.setTotalGarcia(cal.calcularByRequest(request));
		
		
		
		String jsonResponse = json.toJson(response);
        JmsMessage jmsMessageOut = new JmsMessage(jsonResponse, null, "calcular.out");
        System.out.println(jsonResponse);
        jmsSender.send(jmsMessageOut);
		
	}
}

```

# 7. Construir el enviador:

Creamos otro proyecto con las mismas dependencias