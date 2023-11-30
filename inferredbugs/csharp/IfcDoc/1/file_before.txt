// Name:        XmlSerializer.cs
// Description: XML serializer
// Author:      Tim Chipman
// Origination: Work performed for BuildingSmart by Constructivity.com LLC.
// Copyright:   (c) 2017 BuildingSmart International Ltd.
// License:     http://www.buildingsmart-tech.org/legal

using System;
using System.Collections;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations.Schema;
using System.IO;
using System.Linq;
using System.Reflection;
using System.Text;
using System.Runtime.Serialization;
using System.Threading.Tasks;
using System.Xml.Serialization;
using System.Xml;

namespace BuildingSmart.Serialization.Xml
{
	public class XmlSerializer : Serializer
	{
		protected ObjectStore mObjectStore = new ObjectStore();

		public XmlSerializer(Type type) : base(type)
		{
			// get the XML namespace
		}
		public bool UseUniqueIdReferences { get { return mObjectStore.UseUniqueIdReferences; } set { mObjectStore.UseUniqueIdReferences = value; } }

		public override object ReadObject(Stream stream)
		{
			if (stream == null)
				throw new ArgumentNullException("stream");

			// pull it into a memory stream so we can make multiple passes (can't assume it's a file; could be web service)
			//...MemoryStream memstream = new MemoryStream();

			Dictionary<string, object> instances = new Dictionary<string, object>();
			ReadObject(stream, out instances);

			// stash project in empty string key
			object root = null;
			if (instances.TryGetValue(String.Empty, out root))
			{
				return root;
			}

			return null; // could not find the single project object
		}

		/// <summary>
		/// Reads an object graph and provides access to instance identifiers from file.
		/// </summary>
		/// <param name="stream"></param>
		/// <param name="instances"></param>
		/// <returns></returns>
		public object ReadObject(Stream stream, out Dictionary<string, object> instances)
		{
			System.Diagnostics.Debug.WriteLine("!! Reading XML");
			instances = new Dictionary<string, object>();
			
			return ReadContent(stream, instances);
		}

		/// <summary>
		/// 
		/// </summary>
		/// <param name="stream"></param>
		/// <param name="idmap"></param>
		/// <param name="parsefields">True to populate fields; False to load instances only.</param>
		private object ReadContent(Stream stream, Dictionary<string, object> instances)
		{
			object result = null;
			QueuedObjects queuedObjects = new QueuedObjects();
			using (XmlReader reader = XmlReader.Create(stream))
			{
				while (reader.Read())
				{
					switch (reader.NodeType)
					{
						case XmlNodeType.Element:
							if (reader.Name == "ex:iso_10303_28")
							{
								//ReadIsoStep(reader, fixups, instances, inversemap);
							}
							else// if (reader.LocalName.Equals("ifcXML"))
							{
								//ReadPopulation(reader, fixups, instances, inversemap);

								while (reader.Read())
								{
									switch (reader.NodeType)
									{
										case XmlNodeType.Element:
											object o = ReadEntity(reader, instances, reader.LocalName, queuedObjects, false);
											if (result == null && o != null)
												result = o;
											break;

										case XmlNodeType.EndElement:
											return result;
									}
								}

							}
							break;
					}
				}
			}
			return result;
		}

		protected object ReadEntity(XmlReader reader, IDictionary<string, object> instances, string typename, QueuedObjects queuedObjects, bool nestedElementDefinition)
		{
			object o = null;
			Type t = null;
			if (string.IsNullOrEmpty(typename))
				t = GetTypeByName(reader.LocalName);
			else if (string.Compare(typename, "header", true) == 0)
				t = typeof(header);
			else
				t = GetTypeByName(typename);

			if (t != null)
			{
				if (t.IsAbstract)
				{
					reader.MoveToElement();
					while (reader.Read())
					{
						switch (reader.NodeType)
						{
							case XmlNodeType.Element:
								{
									Type localType = this.GetTypeByName(reader.LocalName);
									if (localType != null && localType.IsSubclassOf(t) && !localType.IsAbstract)
										o = ReadEntity(reader, instances, reader.LocalName, queuedObjects, false);
									break;
								}

							case XmlNodeType.Attribute:
								break;

							case XmlNodeType.EndElement:
								//System.Diagnostics.Debug.WriteLine("!!ReadEntity: " + t.Name + "." + reader.LocalName);
								return o;
						}
					}
					//System.Diagnostics.Debug.WriteLine("<<ReadEntity: " + t.Name + "." + reader.LocalName);
					return o;
				}
				//System.Diagnostics.Debug.WriteLine(">>ReadEntity: " + t.Name + "." + reader.LocalName);
				// map instance id if used later
				string sid = reader.GetAttribute("id");

				if (t == this.ProjectType || t.IsSubclassOf(this.ProjectType))
				{
					if (!instances.TryGetValue(String.Empty, out o))
					{
						o = instances[String.Empty] = FormatterServices.GetUninitializedObject(t); // stash project using blank index
						if (!string.IsNullOrEmpty(sid))
							instances[sid] = o;
					}
				}
				else if (!String.IsNullOrEmpty(sid) && !instances.TryGetValue(sid, out o))
				{
					o = FormatterServices.GetUninitializedObject(t);
					instances.Add(sid, o);
				}
				if (o == null)
				{
					o = FormatterServices.GetUninitializedObject(t);
					if (!String.IsNullOrEmpty(sid))
					{
						instances.Add(sid, o);
					}
				}
				if (!string.IsNullOrEmpty(sid))
				{
					queuedObjects.DeQueue(sid, o);
				}
				// ensure all lists/sets are instantiated
				Initialize(o, t);


				// read attribute properties
				for (int i = 0; i < reader.AttributeCount; i++)
				{
					reader.MoveToAttribute(i);
					if (!reader.LocalName.Equals("id"))
					{
						string match = reader.LocalName;
						PropertyInfo f = GetFieldByName(t, match);
						if (f != null)
						{
							ReadValue(reader, o, f, f.PropertyType);
						}
					}
				}

				reader.MoveToElement();
				// now read attributes or end of entity
				if (reader.IsEmptyElement)
				{
					//	System.Diagnostics.Debug.WriteLine("<<ReadEntity: " + t.Name + "." + reader.LocalName);
					return o;
				}
			}
			bool isNested = reader.AttributeCount == 0 && nestedElementDefinition;
			while (reader.Read())
			{
				switch (reader.NodeType)
				{
					case XmlNodeType.Element:
						{
							if (isNested)
							{
								if(t == null && !string.IsNullOrEmpty(reader.LocalName))
								{
									o = ReadEntity(reader, instances, reader.LocalName, queuedObjects, false);
									break;
								}
								else if (string.Compare(reader.LocalName, t.Name) == 0)
								{
									o = ReadEntity(reader, instances, typename, queuedObjects, false);
									break;
								}
								else
								{
									Type localType = this.GetNonAbstractTypeByName(reader.LocalName);
									if (localType != null && localType.IsSubclassOf(t))
									{
										o = ReadEntity(reader, instances, reader.LocalName, queuedObjects, false);
										break;
									}
								}
							}
							ReadElement(reader, o, instances, queuedObjects);
							break;
						}

					case XmlNodeType.Attribute:
						break;

					case XmlNodeType.EndElement:
						//System.Diagnostics.Debug.WriteLine("!!ReadEntity: " + t.Name + "." + reader.LocalName);
						return o;
				}
			}
			//System.Diagnostics.Debug.WriteLine("<<ReadEntity: " + t.Name + "." + reader.LocalName);


			return o;
		}

		/// <summary>
		/// 
		/// </summary>
		/// <param name="reader"></param>
		/// <param name="o"></param>
		/// <param name="instances"></param>
		protected void ReadElement(XmlReader reader, object o, IDictionary<string, object> instances, QueuedObjects queuedObjects)
		{
			if (o == null)
				throw new ArgumentNullException("o");

			//System.Diagnostics.Debug.WriteLine(">>ReadElement: " + o.GetType().Name + "." + reader.LocalName);

			// read attribute of object
			string match = reader.LocalName;
			PropertyInfo f = GetFieldByName(o.GetType(), match);

			// inverse
			if (f == null)
			{
				f = GetInverseByName(o.GetType(), match);
			}

			if (f == null)
			{

			System.Diagnostics.Debug.WriteLine("XX XMLSerializer: " + o.GetType().Name + "::" + reader.LocalName + " attribute name does not exist.");
				return;
			}

			// read attribute properties
			string reftype = null;

			if (!f.PropertyType.IsGenericType &&
				!f.PropertyType.IsInterface &&
				f.PropertyType != typeof(DateTime) &&
				f.PropertyType != typeof(string))
			{
				reftype = f.PropertyType.Name;
			}

			// interface, e.g. IfcRepresentation.StyledByItem\IfcStyledItem
			string r = reader.GetAttribute("href");
			if(!string.IsNullOrEmpty(r))
			{
				processReference(r, o, f, instances, queuedObjects);
				return;
			}
			string t = reader.GetAttribute("xsi:type");
			if (!string.IsNullOrEmpty(t))
			{
				reftype = t;
				if (t.Contains(":"))
				{
					string[] parts = t.Split(':');
					if (parts.Length == 2)
					{
						reftype = parts[1];
					}
				}
			}

			if (reftype != null)
			{
				this.ReadReference(reader, o, f, instances, queuedObjects, reftype);
			}
			else
			{
				// now read object(s) of attribute -- multiple indicates list
				if (!reader.IsEmptyElement)
				{
					while (reader.Read())
					{
						switch (reader.NodeType)
						{
							case XmlNodeType.Attribute:
								if(reader.LocalName == "href")
									processReference(reader.Value, o, f, instances, queuedObjects);
								break;	
							case XmlNodeType.Element:
								ReadReference(reader, o, f, instances, queuedObjects, reader.LocalName);
								break;

							case XmlNodeType.Text:
							case XmlNodeType.CDATA:
								ReadValue(reader, o, f, null);
								break;

							case XmlNodeType.EndElement:
							//	System.Diagnostics.Debug.WriteLine("!!ReadElement: " + o.GetType().Name + "." + reader.LocalName + " " + o.ToString());
								return;
						}
					}
				}
			}

			//System.Diagnostics.Debug.WriteLine("<<ReadElement: " + o.GetType().Name + "." + reader.LocalName + " " + o.ToString());

		}

		/// <summary>
		/// Reads a reference or qualified value.
		/// </summary>
		/// <param name="reader"></param>
		/// <param name="o"></param>
		/// <param name="f"></param>
		private void ReadReference(XmlReader reader, object o, PropertyInfo f, IDictionary<string, object> instances, QueuedObjects queuedObjects, string typename)
		{
			if (typename.EndsWith("-wrapper"))
			{
				typename = typename.Substring(0, typename.Length - 8);
			}

			Type vt = GetTypeByName(typename);
			if (reader.Name == "ex:double-wrapper")
			{
				// drill in
				if (!reader.IsEmptyElement)
				{
					while (reader.Read())
					{
						switch (reader.NodeType)
						{
							case XmlNodeType.Text:
								ReadValue(reader, o, f, typeof(double));
								break;

							case XmlNodeType.EndElement:
								return;
						}
					}
				}
			}
			else if (vt != null)
			{
				Type et = f.PropertyType;
				if (typeof(IEnumerable).IsAssignableFrom(f.PropertyType))
				{
					et = f.PropertyType.GetGenericArguments()[0];
				}

				if (vt.IsValueType)
				{
					// drill in
					if (!reader.IsEmptyElement)
					{
						bool hasvalue = false;
						while (reader.Read())
						{
							switch (reader.NodeType)
							{
								case XmlNodeType.Text:
								case XmlNodeType.CDATA:
									if (ReadValue(reader, o, f, vt))
										return;
									hasvalue = true;
									break;

								case XmlNodeType.EndElement:
									if (!hasvalue)
									{
										ReadValue(reader, o, f, vt);
									}
									return;
							}
						}
					}
				}
				else
				{
					// reference
					string r = reader.GetAttribute("href");
					if (!string.IsNullOrEmpty(r))
					{
						object value = null;
						if (instances.TryGetValue(r, out value))
						{
							LoadEntityValue(o, f, value);
						}
						else
						{
							queuedObjects.Queue(r, o, f);
						}
					}
					else
					{
						// embedded entity definition
						object v = this.ReadEntity(reader, instances, typename, queuedObjects, true);
						LoadEntityValue(o, f, v);
					}
				}
			}
		}

		private void LoadCollectionValue(IEnumerable list, object v)
		{
			if (list == null)
				return;

			Type typeCollection = list.GetType();

			try
			{
				MethodInfo methodAdd = typeCollection.GetMethod("Add");
				methodAdd.Invoke(list, new object[] { v }); // perf!!
			}
			catch (Exception)
			{
				// could be type that changed and is no longer compatible with schema -- try to keep going
			}
		}
		private bool processReference(string sid, object o, PropertyInfo f, IDictionary<string, object> instances, QueuedObjects queuedObjects)
		{
			if (string.IsNullOrEmpty(sid))
				return false;

			object encounteredObject = null;
			if (instances.TryGetValue(sid, out encounteredObject))
			{
				LoadEntityValue(o, f, encounteredObject);
			}
			else
			{
				queuedObjects.Queue(sid, o, f);
			}
			return true;
		}
		protected void LoadEntityValue(object o, PropertyInfo f, object v)
		{
			if (v == null)
				return;

			//System.Diagnostics.Debug.WriteLine(">>LoadValue: " + o.GetType().Name + "." + f.Name + " " + v.ToString());
			if (!f.PropertyType.IsValueType &&
				typeof(IEnumerable).IsAssignableFrom(f.PropertyType) &&
				f.PropertyType.IsGenericType &&
				f.PropertyType.GetGenericArguments()[0].IsInstanceOfType(v))
			{
				if (f.IsDefined(typeof(InversePropertyAttribute), false))//f.FieldType.GetGenericTypeDefinition() == typeof(SInverse<>))
				{
					InversePropertyAttribute[] attrs = (InversePropertyAttribute[])f.GetCustomAttributes(typeof(InversePropertyAttribute), false);

					// set the direct field, map to inverse
					PropertyInfo fieldDirect = GetFieldByName(v.GetType(), attrs[0].Property);
					if (IsEntityCollection(fieldDirect.PropertyType))
					{
						System.Collections.IEnumerable list = fieldDirect.GetValue(v) as System.Collections.IEnumerable;
						try
						{
							Type typeCollection = this.GetCollectionInstanceType(fieldDirect.PropertyType);
							MethodInfo methodAdd = typeCollection.GetMethod("Add");
							methodAdd.Invoke(list, new object[] { o }); // perf!!
						}
						catch (Exception e)
						{
							// could be type that changed and is no longer compatible with schema -- try to keep going
						}
					}
					else
					{
						// single object
						fieldDirect.SetValue(v, o);
					}

					// also add to inverse

					// allocate collection as needed
					System.Collections.IEnumerable listInv = (System.Collections.IEnumerable)f.GetValue(o);
					if (listInv == null)
					{
						Type typeCollection = this.GetCollectionInstanceType(f.PropertyType);
						listInv = (System.Collections.IEnumerable)System.Activator.CreateInstance(typeCollection);
						f.SetValue(o, listInv);
					}

					// add to inverse collection
					try
					{
						Type typeCollection = this.GetCollectionInstanceType(f.PropertyType);
						MethodInfo methodAdd = typeCollection.GetMethod("Add");
						methodAdd.Invoke(listInv, new object[] { v }); // perf!!
					}
					catch (Exception e)
					{
						// could be type that changed and is no longer compatible with schema -- try to keep going
					}

				}
				else
				{
					// add to set/list of values
					IEnumerable list = f.GetValue(o) as IEnumerable;
					if (list != null)
					{
						try
						{
							Type typeCollection = this.GetCollectionInstanceType(f.PropertyType);
							MethodInfo methodAdd = typeCollection.GetMethod("Add");
							methodAdd.Invoke(list, new object[] { v }); // perf!!
						}
						catch (Exception e)
						{
							// could be type that changed and is no longer compatible with schema -- try to keep going
						}

						UpdateInverse(o, f, v);
					}
				}
			}
			else
			{
				if (!f.PropertyType.IsInstanceOfType(v))
				{
					//this.Log("IFCXML: #" + o.OID + ": " + o.GetType().Name + "." + f.Name + ": '" + v.GetType().Name + "' type is incompatible.");
					return;
				}

				// set value
				f.SetValue(o, v);
				this.UpdateInverse(o, f, v);
			}

		}

		/// <summary>
		/// Reads a value
		/// </summary>
		/// <param name="reader">The xml reader</param>
		/// <param name="o">The entity</param>
		/// <param name="f">The field</param>
		/// <param name="ft">Optional explicit type, or null to use field type.</param>
		private bool ReadValue(XmlReader reader, object o, PropertyInfo f, Type ft)
		{
			bool endelement = false;

			if (ft == null)
			{
				ft = f.PropertyType;
			}

			if (ft.IsGenericType && ft.GetGenericTypeDefinition() == typeof(Nullable<>))
			{
				// special case for Nullable types
				ft = ft.GetGenericArguments()[0];
			}

			object v = null;
			if (ft.IsEnum)
			{
				FieldInfo enumfield = ft.GetField(reader.Value, BindingFlags.IgnoreCase | BindingFlags.Public | System.Reflection.BindingFlags.Static);
				if (enumfield != null)
				{
					v = enumfield.GetValue(null);
				}
			}
			else if ( ft == typeof(DateTime) || ft == typeof(string) || ft == typeof(byte[]))
			{
				v = ParsePrimitive(reader.Value, ft);
			}
			else if (ft.IsValueType)
			{
				// defined type -- get the underlying field
				PropertyInfo[] fields = ft.GetProperties(BindingFlags.Instance | BindingFlags.Public); //perf: cache this
				if (fields.Length == 1)
				{
					PropertyInfo fieldValue = fields[0];
					object primval = ParsePrimitive(reader.Value, fieldValue.PropertyType);
					v = Activator.CreateInstance(ft);
					fieldValue.SetValue(v, primval);
				}
			}
			else if (IsEntityCollection(ft))
			{
				// IfcCartesianPoint.Coordinates

				Type typeColl = GetCollectionInstanceType(ft);
				v = System.Activator.CreateInstance(typeColl);

				Type typeElem = ft.GetGenericArguments()[0];
				PropertyInfo propValue = typeElem.GetProperty("Value");

				if (propValue != null)
				{
					string[] elements = reader.Value.Split(new char[] { ' ' }, StringSplitOptions.RemoveEmptyEntries);

					IEnumerable list = (IEnumerable)v;
					foreach (string elem in elements)
					{
						object elemv = Activator.CreateInstance(typeElem);
						object primv = ParsePrimitive(elem, propValue.PropertyType);
						propValue.SetValue(elemv, primv);
						LoadCollectionValue(list, elemv);
					}
				}
			}

			LoadEntityValue(o, f, v);

			return endelement;
		}

		private static object ParsePrimitive(string readervalue, Type type)
		{
			object value = null;
			if (typeof(Int64) == type)
			{
				// INTEGER
				value = ParseInteger(readervalue);
			}
			else if (typeof(Int32) == type)
			{
				value = (Int32)ParseInteger(readervalue);
			}
			else if (typeof(Double) == type)
			{
				// REAL
				value = ParseReal(readervalue);
			}
			else if (typeof(Single) == type)
			{
				value = (Single)ParseReal(readervalue);
			}
			else if (typeof(Boolean) == type)
			{
				// BOOLEAN
				value = ParseBoolean(readervalue);
			}
			else if (typeof(String) == type)
			{
				// STRING
				value = readervalue.Trim();
			}
			else if (typeof(DateTime) == type)
			{
				DateTime dtVal;
				if (DateTime.TryParse(readervalue, out dtVal))
				{
					value = dtVal;
				}
			}
			else if (typeof(byte[]) == type)
			{
				value = ParseBinary(readervalue);
			}

			return value;
		}

		private static bool ParseBoolean(string strval)
		{
			bool iv;
			if (Boolean.TryParse(strval, out iv))
			{
				return iv;
			}

			return false;
		}

		private static Int64 ParseInteger(string strval)
		{
			long iv;
			if (Int64.TryParse(strval, out iv))
			{
				return iv;
			}

			return 0;
		}

		private static Double ParseReal(string strval)
		{
			double iv;
			if (Double.TryParse(strval, out iv))
			{
				return iv;
			}

			return 0.0;
		}

		/// <summary>
		/// Writes an object graph to a stream formatted xml.
		/// </summary>
		/// <param name="stream">The stream to write.</param>
		/// <param name="root">The root object to write (typically IfcProject)</param>
		public override void WriteObject(Stream stream, object root)
		{
			if (stream == null)
				throw new ArgumentNullException("stream");

			if (root == null)
				throw new ArgumentNullException("root");

			// pass 1: (first time ever encountering for serialization) -- determine which entities require IDs -- use a null stream
			int nextID = 0;
			int indent = 0;
			StreamWriter writer = new StreamWriter(Stream.Null);
			Queue<object> queue = new Queue<object>();
			queue.Enqueue(root);
			while (queue.Count > 0)
			{
				object ent = queue.Dequeue();
				if (string.IsNullOrEmpty(mObjectStore.EncounteredId(ent)))
				{
					this.WriteEntity(writer, ref indent, ent, new HashSet<string>(), queue, true, ref nextID);
				}
			}
			// pass 2: write to file -- clear save map; retain ID map
			mObjectStore.ClearEncounterd();
			writeObject(stream, root, new HashSet<string>(), new Queue<object>(), false, ref nextID);
		}
		internal protected void writeObject(Stream stream, object root, HashSet<string> propertiesToIgnore, ref int nextID)
		{
			Queue<object> queue = new Queue<object>();
			queue.Enqueue(root);
			writeObject(stream, root, propertiesToIgnore, queue, false, ref nextID);
		}
		private void writeObject(Stream stream, object root, HashSet<string> propertiesToIgnore, Queue<object> queue, bool isIdPass, ref int nextID)
		{ 
			int indent = 0;
			StreamWriter writer = new StreamWriter(stream);

			this.WriteHeader(writer);

			bool rootdelim = false;
			queue.Enqueue(root);
			while (queue.Count > 0)
			{
				// insert delimeter after first root object
				if (rootdelim)
				{
					this.WriteRootDelimeter(writer);
				}
				rootdelim = true;

				object ent = queue.Dequeue();
				if(string.IsNullOrEmpty(mObjectStore.EncounteredId(ent)))
				{
					this.WriteEntity(writer, ref indent, ent, propertiesToIgnore, queue, isIdPass, ref nextID);
				}
			}

			this.WriteFooter(writer);

			writer.Flush();
		}

		protected virtual void WriteHeader(StreamWriter writer)
		{
			string header = "<?xml version=\"1.0\" encoding=\"utf-8\"?>";

			string schema = "<ifc:ifcXML xsi:schemaLocation=\"" + this.BaseURI + " " + this.Schema + ".xsd\" " +
				"xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" " +
				"xmlns:ifc=\"" + this.BaseURI + "\" " +
				"xmlns=\"" + this.BaseURI + "\">";

			writer.WriteLine(header);
			writer.WriteLine(schema);

			// writer header info
			header h = new header();
			h.time_stamp = DateTime.UtcNow;
			h.preprocessor_version = this.Preprocessor;
			h.originating_system = this.Application;
			int indent = 0, nextID = 0;
			this.WriteEntity(writer, ref indent, h, new HashSet<string>(), new Queue<object>(), false, ref nextID);
		}


		protected virtual void WriteFooter(StreamWriter writer)
		{
			string footer = "</ifc:ifcXML>";
			writer.WriteLine(footer);
			writer.Write("\r\n\r\n");
		}

		protected virtual void WriteRootDelimeter(StreamWriter writer)
		{
		}

		protected virtual void WriteCollectionStart(StreamWriter writer, ref int indent)
		{
		}

		protected virtual void WriteCollectionDelimiter(StreamWriter writer, int indent)
		{
		}

		protected virtual void WriteCollectionEnd(StreamWriter writer, ref int indent)
		{
		}

		protected virtual void WriteEntityStart(StreamWriter writer, ref int indent)
		{
		}

		protected virtual void WriteEntityEnd(StreamWriter writer, ref int indent)
		{
		}

		private void WriteEntity(StreamWriter writer, ref int indent, object o, HashSet<string> propertiesToIgnore, Queue<object> queue, bool isIdPass, ref int nextID)
		{
			// sanity check
			if (indent > 100)
			{
				return;
			}

			if (o == null)
				return;

			Type t = o.GetType();

			this.WriteStartElementEntity(writer, ref indent, t.Name);
			bool close = this.WriteEntityAttributes(writer, ref indent, o, propertiesToIgnore, queue, isIdPass, ref nextID);
			if (close)
			{
				this.WriteEndElementEntity(writer, ref indent, t.Name);
			}
			else
			{
				this.WriteCloseElementEntity(writer, ref indent);
			}
		}

		/// <summary>
		/// Terminates the opening tag, to allow for sub-elements to be written
		/// </summary>
		protected virtual void WriteOpenElement(StreamWriter writer) { WriteOpenElement(writer, true); }
		protected virtual void WriteOpenElement(StreamWriter writer, bool newLine)
		{
			// end opening tag
			if (newLine)
				writer.WriteLine(">");
			else
				writer.Write(">");
		}

		/// <summary>
		/// Terminates the opening tag, with no subelements
		/// </summary>
		protected virtual void WriteCloseElementEntity(StreamWriter writer, ref int indent)
		{
			writer.WriteLine(" />");
			indent--;
		}

		protected virtual void WriteCloseElementAttribute(StreamWriter writer, ref int indent)
		{
			this.WriteCloseElementEntity(writer, ref indent);
		}

		protected virtual void WriteStartElementEntity(StreamWriter writer, ref int indent, string name)
		{
			this.WriteIndent(writer, indent);
			writer.Write("<" + name);
			indent++;
		}

		protected virtual void WriteStartElementAttribute(StreamWriter writer, ref int indent, string name)
		{
			this.WriteStartElementEntity(writer, ref indent, name);
		}

		protected virtual void WriteEndElementEntity(StreamWriter writer, ref int indent, string name)
		{
			indent--;

			this.WriteIndent(writer, indent);
			writer.Write("</");
			writer.Write(name);
			writer.WriteLine(">");
		}

		protected virtual void WriteEndElementAttribute(StreamWriter writer, ref int indent, string name)
		{
			WriteEndElementEntity(writer, ref indent, name);
		}

		protected virtual void WriteIdentifier(StreamWriter writer, int indent, string id)
		{
			// record id, and continue to write out all attributes (works properly on second pass)
			writer.Write(" id=\"");
			writer.Write(id);
			writer.Write("\"");
		}

		protected virtual void WriteReference(StreamWriter writer, int indent, string id)
		{
			writer.Write(" xsi:nil=\"true\" href=\"");
			writer.Write(id);
			writer.Write("\"");
		}

		protected virtual void WriteType(StreamWriter writer, int indent, string type)
		{
			writer.Write(" xsi:type=\"");
			writer.Write(type);
			writer.Write("\"");
		}

		protected virtual void WriteTypedValue(StreamWriter writer, ref int indent, string type, string encodedvalue)
		{
			this.WriteIndent(writer, indent);
			writer.WriteLine("<" + type + "-wrapper>" + encodedvalue + "</" + type + "-wrapper>");
		}

		protected virtual void WriteStartAttribute(StreamWriter writer, int indent, string name)
		{
			writer.Write(" ");
			writer.Write(name);
			writer.Write("=\"");
		}

		protected virtual void WriteEndAttribute(StreamWriter writer)
		{
			writer.Write("\"");
		}

		protected virtual void WriteAttributeDelimiter(StreamWriter writer)
		{
		}

		protected virtual void WriteAttributeTerminator(StreamWriter writer)
		{
		}

		

		protected static bool IsValueCollection(Type t)
		{
			return t.IsGenericType &&
				typeof(IEnumerable).IsAssignableFrom(t.GetGenericTypeDefinition()) &&
				t.GetGenericArguments()[0].IsValueType;
		}

		
		/// <summary>
		/// Returns true if any elements written (requiring closing tag); or false if not
		/// </summary>
		/// <param name="o"></param>
		/// <returns></returns>
		private bool WriteEntityAttributes(StreamWriter writer, ref int indent, object o, HashSet<string> propertiesToIgnore, Queue<object> queue, bool isIdPass, ref int nextID)
		{
			Type t = o.GetType(), stringType = typeof(String);

			string id = mObjectStore.EncounteredId(o);
			if (!string.IsNullOrEmpty(id))
			{
				// reference existing; return
				this.WriteReference(writer, indent, id);
				return false;
			}
			// give it an ID if needed (first pass)
			// mark as saved
			id = mObjectStore.ProccessId(o, isIdPass, ref nextID);

			if (!string.IsNullOrEmpty(id))
			{
				this.WriteIdentifier(writer, indent, id);
			}

			bool previousattribute = false;

			// write fields as attributes
			IList<PropertyInfo> fields = this.GetFieldsAll(t);
			List<PropertyInfo> elementFields = new List<PropertyInfo>();
			foreach (PropertyInfo f in fields)
			{
				if (f != null) // derived fields are null
				{
					if (propertiesToIgnore.Contains(f.Name))
						continue;
					DocXsdFormatEnum? xsdformat = this.GetXsdFormat(f);
					if (xsdformat == DocXsdFormatEnum.Hidden)
						continue;

					if (f.IsDefined(typeof(DataMemberAttribute)) && (xsdformat == null || xsdformat == DocXsdFormatEnum.Attribute))
					{
						// direct field

						Type ft = f.PropertyType;

						bool isvaluelist = IsValueCollection(ft);
						bool isvaluelistlist = ft.IsGenericType && // e.g. IfcTriangulatedFaceSet.Normals
							typeof(System.Collections.IEnumerable).IsAssignableFrom(ft.GetGenericTypeDefinition()) &&
							IsValueCollection(ft.GetGenericArguments()[0]);

						if (isvaluelistlist || isvaluelist || ft.IsValueType || ft == stringType)
						{
							object v = f.GetValue(o);
							if (v != null)
							{
								if (previousattribute)
								{
									this.WriteAttributeDelimiter(writer);
								}

								previousattribute = true;
								this.WriteStartAttribute(writer, indent, f.Name);

								if (isvaluelistlist)
								{
									ft = ft.GetGenericArguments()[0].GetGenericArguments()[0];
									PropertyInfo fieldValue = ft.GetProperty("Value");
									if (fieldValue != null)
									{
										System.Collections.IList list = (System.Collections.IList)v;
										for (int i = 0; i < list.Count; i++)
										{
											System.Collections.IList listInner = (System.Collections.IList)list[i];
											for (int j = 0; j < listInner.Count; j++)
											{
												if (i > 0 || j > 0)
												{
													writer.Write(" ");
												}

												object elem = listInner[j];
												if (elem != null) // should never be null, but be safe
												{
													elem = fieldValue.GetValue(elem);
													string encodedvalue = System.Security.SecurityElement.Escape(elem.ToString());
													writer.Write(encodedvalue);
												}
											}
										}
									}
									else
									{
										System.Diagnostics.Debug.WriteLine("XXX Error serializing ValueListlist" + o.ToString());
									}
								}
								else if (isvaluelist)
								{
									ft = ft.GetGenericArguments()[0];
									PropertyInfo fieldValue = ft.GetProperty("Value");

									IEnumerable list = (IEnumerable)v;
									int i = 0;
									foreach (object e in list)
									{
										if (i > 0)
										{
											writer.Write(" ");
										}

										if (e != null) // should never be null, but be safe
										{
											object elem = e;
											if (fieldValue != null)
											{
												elem = fieldValue.GetValue(e);
											}

											if (elem is byte[])
											{
												// IfcPixelTexture.Pixels
												writer.Write(SerializeBytes((byte[])elem));
											}
											else
											{
												string encodedvalue = System.Security.SecurityElement.Escape(elem.ToString());
												writer.Write(encodedvalue);
											}
										}

										i++;
									}

								}
								else
								{
									if (ft.IsGenericType && ft.GetGenericTypeDefinition() == typeof(Nullable<>))
									{
										// special case for Nullable types
										ft = ft.GetGenericArguments()[0];
									}

									Type typewrap = null;
									while (ft.IsValueType && !ft.IsPrimitive)
									{
										PropertyInfo fieldValue = ft.GetProperty("Value");
										if (fieldValue != null)
										{
											v = fieldValue.GetValue(v);
											if (typewrap == null)
											{
												typewrap = ft;
											}
											ft = fieldValue.PropertyType;
										}
										else
										{
											break;
										}
									}

									if (ft.IsEnum || ft == typeof(bool))
									{
										v = v.ToString().ToLowerInvariant();
									}

									if (v is System.Collections.IList)
									{
										// IfcCompoundPlaneAngleMeasure
										System.Collections.IList list = (System.Collections.IList)v;
										for (int i = 0; i < list.Count; i++)
										{
											if (i > 0)
											{
												writer.Write(" ");
											}

											object elem = list[i];
											if (elem != null) // should never be null, but be safe
											{
												string encodedvalue = System.Security.SecurityElement.Escape(elem.ToString());
												writer.Write(encodedvalue);
											}
										}
									}
									else if (v != null)
									{
										string encodedvalue = System.Security.SecurityElement.Escape(v.ToString());
										writer.Write(encodedvalue);
									}
								}

								this.WriteEndAttribute(writer);
							}
						}
						else
						{
							elementFields.Add(f);
						}
					}
					else
					{
						elementFields.Add(f);
					}
				}
			}

			if (elementFields.Count > 0)
			{
				bool open = false;

				// write direct object references and lists
				foreach (PropertyInfo f in elementFields)
				{
					if (f != null) // derived attributes are null
					{
						DocXsdFormatEnum? format = GetXsdFormat(f);
						if (format == DocXsdFormatEnum.Element)
						{
							object value = f.GetValue(o);
							if (value != null)
							{
								// for collection is must be non-zero (e.g. IfcProject.IsNestedBy)
								bool showit = true; //...check: always include tag if Attribute (even if zero); hide if Element 
								
								if (value is IEnumerable) // what about IfcProject.RepresentationContexts if empty? include???
								{
									showit = false;
									IEnumerable enumerate = (IEnumerable)value;
									foreach (object check in enumerate)
									{
										showit = true; // has at least one element
											break;
									}
								}
								if (showit)
								{
									if (!open)
									{
										WriteOpenElement(writer);
										open = true;
									}

									if (previousattribute)
									{
										this.WriteAttributeDelimiter(writer);
									}
									previousattribute = true;
									WriteAttribute(writer, ref indent, o, new HashSet<string>(), f, queue, isIdPass, ref nextID);
								}
							}
						}
						else if (f.IsDefined(typeof(DataMemberAttribute)))
						{
							Type ft = f.PropertyType;
							bool isvaluelist = IsValueCollection(ft);
							bool isvaluelistlist = ft.IsGenericType && // e.g. IfcTriangulatedFaceSet.Normals
								typeof(IEnumerable).IsAssignableFrom(ft.GetGenericTypeDefinition()) &&
								IsValueCollection(ft.GetGenericArguments()[0]);
							
							// hide fields where inverse attribute used instead
							if (!ft.IsValueType && !isvaluelist && !isvaluelistlist)
							{
								object value = f.GetValue(o);
								if (value != null)
								{
									bool showit = true;

									if (!f.IsDefined(typeof(System.ComponentModel.DataAnnotations.RequiredAttribute), false) && value is IEnumerable)
									{
										showit = false;
										IEnumerable en = (IEnumerable)value;
										foreach (object sub in en)
										{
											showit = true;
											break;
										}
									}

									if (showit)
									{
										if (!open)
										{
											WriteOpenElement(writer);
											open = true;
										}

										if (previousattribute)
										{
											this.WriteAttributeDelimiter(writer);
										}
										previousattribute = true;

										WriteAttribute(writer, ref indent, o, new HashSet<string>(), f, queue, isIdPass, ref nextID);
									}
								}
							}
						}
						else
						{
							object value = f.GetValue(o);

							// inverse
							// record it for downstream serialization
							if (value is IEnumerable)
							{
								IEnumerable invlist = (IEnumerable)value;
								foreach (object invobj in invlist)
								{
									if (string.IsNullOrEmpty(mObjectStore.EncounteredId(invobj)))
									{
										queue.Enqueue(invobj);
									}
								}
							}
						}
					}
				}

				this.WriteAttributeTerminator(writer);
				return open;
			}
			else
			{
				this.WriteAttributeTerminator(writer);
				return false;
			}
		}

		private void WriteAttribute(StreamWriter writer, ref int indent, object o, HashSet<string> propertiesToIgnore, PropertyInfo f, Queue<object> queue, bool isIdPass, ref int nextID)
		{
			object v = f.GetValue(o);
			if (v == null)
				return;

			this.WriteStartElementAttribute(writer, ref indent, f.Name);

			int zeroIndent = 0;
			Type ft = f.PropertyType;
			PropertyInfo fieldValue = null;
			if (ft.IsValueType)
			{
				if (ft == typeof(DateTime)) // header datetime
				{
					this.WriteOpenElement(writer, false);
					DateTime datetime = (DateTime)v;
					string datetimeiso8601 = datetime.ToString("s", System.Globalization.CultureInfo.InvariantCulture);
					writer.Write(datetimeiso8601);
					indent--; 
					WriteEndElementAttribute(writer, ref zeroIndent, f.Name);
					return;
				}
				fieldValue = ft.GetProperty("Value"); // if it exists for value type
			}
			else if (ft == typeof(string))
			{
				this.WriteOpenElement(writer, false);
				string strval = System.Security.SecurityElement.Escape((string)v);
				writer.Write(strval);
				indent--;
				WriteEndElementAttribute(writer, ref zeroIndent, f.Name);
				return;
			}
			else if (ft == typeof(byte[]))
			{
				this.WriteOpenElement(writer, false);
				string strval = SerializeBytes((byte[])v); 
				writer.Write(strval);
				indent--;
				WriteEndElementAttribute(writer, ref zeroIndent, f.Name);
				return;
			}
			DocXsdFormatEnum? format = GetXsdFormat(f);
			if (format == null || format != DocXsdFormatEnum.Attribute || f.Name.Equals("InnerCoordIndices")) //hackhack -- need to resolve...
			{
				this.WriteOpenElement(writer);
			}

			if (IsEntityCollection(ft))
			{
				IEnumerable list = (IEnumerable)v;

				// for nested lists, flatten; e.g. IfcBSplineSurfaceWithKnots.ControlPointList
				if (typeof(IEnumerable).IsAssignableFrom(ft.GetGenericArguments()[0]))
				{
					// special case
					if (f.Name.Equals("InnerCoordIndices")) //hack
					{
						foreach (System.Collections.IEnumerable innerlist in list)
						{
							string entname = "Seq-IfcPositiveInteger-wrapper"; // hack
							this.WriteStartElementEntity(writer, ref indent, entname);
							this.WriteOpenElement(writer);
							foreach (object e in innerlist)
							{
								object ev = e.GetType().GetField("Value").GetValue(e);

								writer.Write(ev.ToString());
								writer.Write(" ");
							}
							writer.WriteLine();
							this.WriteEndElementEntity(writer, ref indent, entname);
						}
						WriteEndElementAttribute(writer, ref indent, f.Name);
						return;
					}

					ArrayList flatlist = new ArrayList();
					foreach (IEnumerable innerlist in list)
					{
						foreach (object e in innerlist)
						{
							flatlist.Add(e);
						}
					}

					list = flatlist;
				}

				// required if stated or if populated.....

				foreach (object e in list)
				{
					// if collection is non-zero and contains entity instances
					if (e != null && !e.GetType().IsValueType && !(e is string) && !(e is System.Collections.IEnumerable))
					{
						this.WriteCollectionStart(writer, ref indent);
					}
					break;
				}

				bool needdelim = false;
				foreach (object e in list)
				{
					if (e != null) // could be null if buggy file -- not matching schema
					{
						if (e is IEnumerable)
						{
							IEnumerable listInner = (IEnumerable)e;
							foreach (object oinner in listInner)//j = 0; j < listInner.Count; j++)
							{
								object oi = oinner;//listInner[j];

								Type et = oi.GetType();
								while (et.IsValueType && !et.IsPrimitive)
								{
									PropertyInfo fieldColValue = et.GetProperty("Value");
									if (fieldColValue != null)
									{
										oi = fieldColValue.GetValue(oi);
										et = fieldColValue.PropertyType;
									}
									else
									{
										break;
									}
								}

								// write each value in sequence with spaces delimiting
								string sval = oi.ToString();
								writer.Write(sval);
								writer.Write(" ");
							}
						}
						else if (!e.GetType().IsValueType && !(e is string)) // presumes an entity
						{
							if (needdelim)
							{
								this.WriteCollectionDelimiter(writer, indent);
							}

							if (format != null && format == DocXsdFormatEnum.Attribute)
							{
								// only one item, e.g. StyledByItem\IfcStyledItem
								this.WriteEntityStart(writer, ref indent);
								bool closeelem = this.WriteEntityAttributes(writer, ref indent, e, propertiesToIgnore, queue, isIdPass, ref nextID);
								if (!closeelem)
								{
									this.WriteCloseElementAttribute(writer, ref indent);
									/////?????return;//TWC:20180624
								}
								else
								{
									this.WriteEntityEnd(writer, ref indent);
								}
								break; // if more items, skip them -- buggy input data; no way to encode
							}
							else
							{
								this.WriteEntity(writer, ref indent, e, propertiesToIgnore, queue, isIdPass, ref nextID);
							}

							needdelim = true;
						}
						else
						{
							// if flat-list (e.g. structural load Locations) or list of strings (e.g. IfcPostalAddress.AddressLines), must wrap
							this.WriteValueWrapper(writer, ref indent, e);
						}
					}
				}

				foreach (object e in list)
				{
					if (e != null && !e.GetType().IsValueType && !(e is string))
					{
						this.WriteCollectionEnd(writer, ref indent);
					}
					break;
				}
			} // otherwise if not collection...
			else if (ft.IsInterface && v is ValueType)
			{
				this.WriteValueWrapper(writer, ref indent, v);
			}
			else if (fieldValue != null) // must be IfcBinary -- but not DateTime or other raw primitives
			{
				v = fieldValue.GetValue(v);
				if (v is byte[])
				{
					this.WriteOpenElement(writer);

					// binary data type - we don't support anything other than 8-bit aligned, though IFC doesn't either so no point in supporting extraBits
					byte[] bytes = (byte[])v;

					StringBuilder sb = new StringBuilder(bytes.Length * 2);
					for (int i = 0; i < bytes.Length; i++)
					{
						byte b = bytes[i];
						sb.Append(HexChars[b / 0x10]);
						sb.Append(HexChars[b % 0x10]);
					}
					v = sb.ToString();
					writer.WriteLine(v);
				}
			}
			else
			{
				if (format != null && format == DocXsdFormatEnum.Attribute)
				{
					this.WriteEntityStart(writer, ref indent);

					Type vt = v.GetType();
					if (ft != vt)
					{
						this.WriteType(writer, indent, vt.Name);
					}

					bool closeelem = this.WriteEntityAttributes(writer, ref indent, v, new HashSet<string>(), queue, isIdPass, ref nextID);

					if (!closeelem)
					{
						this.WriteCloseElementEntity(writer, ref indent);
						return;
					}

					this.WriteEntityEnd(writer, ref indent);
				}
				else
				{
					// if rooted, then check if we need to use reference; otherwise embed
					this.WriteEntity(writer, ref indent, v, new HashSet<string>(), queue, isIdPass, ref nextID);
				}
			}

			WriteEndElementAttribute(writer, ref indent, f.Name);
		}

		private void WriteValueWrapper(StreamWriter writer, ref int indent, object v)
		{
			Type vt = v.GetType();
			PropertyInfo fieldValue = vt.GetProperty("Value");
			while (fieldValue != null)
			{
				v = fieldValue.GetValue(v);
				if (v != null)
				{
					Type wt = v.GetType();
					if (wt.IsEnum || wt == typeof(bool))
					{
						v = v.ToString().ToLowerInvariant();
					}

					fieldValue = wt.GetProperty("Value");
				}
				else
				{
					fieldValue = null;
				}
			}

			string encodedvalue = String.Empty;
			if (v is IEnumerable && !(v is string))
			{
				// IfcIndexedPolyCurve.Segments
				IEnumerable list = (IEnumerable)v;
				StringBuilder sb = new StringBuilder();
				foreach (object o in list)
				{
					if (sb.Length > 0)
					{
						sb.Append(" ");
					}

					PropertyInfo fieldValueInner = o.GetType().GetProperty("Value");
					if (fieldValueInner != null)
					{
						//...todo: recurse for multiple levels of indirection, e.g. 
						object vInner = fieldValueInner.GetValue(o);
						sb.Append(vInner.ToString());
					}
					else
					{
						sb.Append(o.ToString());
					}
				}

				encodedvalue = sb.ToString();
			}
			else if (v != null)
			{
				encodedvalue = System.Security.SecurityElement.Escape(v.ToString());
			}

			this.WriteTypedValue(writer, ref indent, vt.Name, encodedvalue);
		}

		protected DocXsdFormatEnum? GetXsdFormat(PropertyInfo field)
		{
			// direct fields marked ignore are ignored
			if (field.IsDefined(typeof(XmlIgnoreAttribute)))
				return DocXsdFormatEnum.Hidden;

			if (field.IsDefined(typeof(XmlAttributeAttribute)))
				return null;

			XmlElementAttribute attrElement = field.GetCustomAttribute<XmlElementAttribute>();
			if (attrElement != null)
			{
				//if (!String.IsNullOrEmpty(attrElement.ElementName))
				//{
					return DocXsdFormatEnum.Element; // tag according to attribute AND element name
				//}
				//else
				//{
				//	return DocXsdFormatEnum.Attribute; // tag according to attribute name
				//}
			}

			// inverse fields not marked with XmlElement are ignored
			if (attrElement == null && field.IsDefined(typeof(InversePropertyAttribute)))
				return DocXsdFormatEnum.Hidden;

			return null; //?
		}

		protected enum DocXsdFormatEnum
		{
			Hidden = 1,//IfcDoc.Schema.CNF.exp_attribute.no_tag,    // for direct attribute, don't include as inverse is defined instead
			Attribute = 2,//IfcDoc.Schema.CNF.exp_attribute.attribute_tag, // represent as attribute
			Element = 3,//IfcDoc.Schema.CNF.exp_attribute.double_tag,   // represent as element
		}

		protected internal class QueuedObjects
		{
			private Dictionary<string, QueuedObject> queued = new Dictionary<string, QueuedObject>();

			internal void Queue(string sid, object o, PropertyInfo propertyInfo)
			{
				QueuedObject queuedObject = null;
				if (!queued.TryGetValue(sid, out queuedObject))
					queuedObject = queued[sid] = new QueuedObject();
				queuedObject.Queue(o, propertyInfo);
			}
			internal void DeQueue(string sid, object value)
			{
				QueuedObject queuedObject = null;
				if(queued.TryGetValue(sid, out queuedObject))
				{
					queuedObject.Dequeue(value);
					queued.Remove(sid);
				}
			}
		}
		protected internal class QueuedObject
		{
			private List<Tuple<object,PropertyInfo>> attributes = new List<Tuple<object,PropertyInfo>>();

			internal void Queue(object o, PropertyInfo propertyInfo)
			{
				attributes.Add(new Tuple<object, PropertyInfo>(o,propertyInfo));
			}
			
			internal void Dequeue(object value)
			{
				foreach (Tuple<object, PropertyInfo> tuple in attributes)
					tuple.Item2.SetValue(tuple.Item1, value);
			}
		}

		protected internal class ObjectStore
		{
			internal bool UseUniqueIdReferences { get; set; } = false;
			private Dictionary<object, string> IdMap = new Dictionary<object, string>();
			private Dictionary<object, string> EncounteredObjects = new Dictionary<object, string>();
			private Dictionary<object, string> IdPassObjects = new Dictionary<object, string>();

			internal ObjectStore() { }

			private string getUniqueId(object o)
			{
				Type ot = o.GetType();
				PropertyInfo propertyInfo = ot.GetProperty("id", typeof(string));
				if (propertyInfo == null)
					propertyInfo = ot.GetProperty("UniqueId", typeof(string));
				if (propertyInfo == null)
					propertyInfo = ot.GetProperty("GlobalId", typeof(string));
				if (propertyInfo != null)
				{
					object obj = propertyInfo.GetValue(o);
					if (obj != null)
						return obj.ToString();
				}
				return "";
			}
			internal protected string createUniqueId(object o, ref int nextID)
			{
				string uniqueId = "";
				if (IdMap.TryGetValue(o, out uniqueId))
					return uniqueId;
				if (UseUniqueIdReferences)
				{
					uniqueId = getUniqueId(o);
				}
				if (string.IsNullOrEmpty(uniqueId))
				{
					nextID++;
					return "i" + nextID;
				}
				IdMap[o] = uniqueId;
				return uniqueId;
			}

			internal string MarkEncountered(Object obj, ref int nextId)
			{
				string id = createUniqueId(obj, ref nextId);
				EncounteredObjects[obj] = id;
				return id;
			}
			internal string EncounteredId(object obj)
			{
				string id = "";
				if (EncounteredObjects.TryGetValue(obj, out id))
					return id;
				return null;
			}
			internal string ProccessId(object obj, bool isIdPass, ref int nextID)
			{
				string id = "";
				if (isIdPass)
				{
					if (IdPassObjects.TryGetValue(obj, out id))
					{
						if (string.IsNullOrEmpty(id))
						{
							return IdPassObjects[obj] = createUniqueId(obj, ref nextID);
						}
					}
					else
						return IdPassObjects[obj] = "";
				}
				else if (UseUniqueIdReferences)
				{
					string uniqueId = getUniqueId(obj);
					if(!string.IsNullOrEmpty(uniqueId))
					{
						return EncounteredObjects[obj] = uniqueId;
					}
				}
				return "";
			}
			internal void RemoveEncountered(object obj)
			{
				EncounteredObjects.Remove(obj);
			}
			internal void ClearEncounterd() { EncounteredObjects.Clear(); }
		}
	}

}


